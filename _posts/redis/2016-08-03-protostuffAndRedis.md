---
layout : post
title :   Protostuff序列化实现对象在redis中缓存
category : redis
tagline: ""
date : 2016-08-03
tags : [redis]
---

-----

#### 一、protostuff介绍

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Google 的protobuf是一个优秀的序列化工具，跨语言、快速、序列化后体积小。protobuf的一个缺点是需要数据结构的预编译过程，首先要编写.proto格式的配置文件，再通过protobuf提供的工具生成各种语言响应的代码。由于java具有反射和动态代码生成的能力，这个预编译过程不是必须的，可以在代码执行时来实现。protostuff基于Google protobuf，但是提供了更多的功能和更简易的用法。其中，protostuff-runtime实现了无需预编译对java bean进行protobuf序列化/反序列化的能力。
Protostuff是一个开源的、基于Java语言的序列化库，它内建支持向前向后兼容（模式演进）和验证功能。

----------

#### 二、redis介绍
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 是完全开源免费的，遵守BSD协议，是一个高性能的key-value数据库。其最大的优势:性能极高 – Redis能读的速度是110000次/s,写的速度是81000次/s 。序列化的意义在于信息的交换和存储，通常会和io、持久化、rmi技术有关,本文主要将如何将Protostuff序列化对象缓存在redis。

------

#### 三、Maven项目,引入redis和protostuff依赖

```xml
	<!--redis客户端:Jedis-->
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.7.3</version>
        </dependency>
        <!--protostuff序列化依赖-->
        <dependency>
            <groupId>com.dyuproject.protostuff</groupId>
            <artifactId>protostuff-core</artifactId>
            <version>1.0.8</version>
        </dependency>
        <dependency>
            <groupId>com.dyuproject.protostuff</groupId>
            <artifactId>protostuff-runtime</artifactId>
            <version>1.0.8</version>
        </dependency>
```

#### 四、RedisDao实现缓存redis对象

##### 1、RedisDao.java code


```java
public class RedisDao {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    private JedisPool jedisPool;

    public RedisDao(String ip, int port, int timeout,String auth) {
        JedisPoolConfig config = new JedisPoolConfig();
        jedisPool = new JedisPool(config, ip, port, timeout, auth);
    }
    //自定义schema
    private RuntimeSchema<Product> schema = RuntimeSchema.createFrom(Product.class);

    /**
     * 从redis缓存中查出序列化的对象，并反序列化出来
     *
     * @return 实体类
     */
    public Product getProduct(long productId) {
        //redis操作逻辑
        try {
            Jedis jedis = jedisPool.getResource();
            try {
                String key = "product:" + productId;
                //并没有实现内部序列化操作
                //get->byte[] -> 反序列化 ->Object(Product)
                //采用自定义序列化
                //protostuff: pojo
                byte[] bytes = jedis.get(key.getBytes());
                if (null != bytes) {
                    //空对象
                    Product product = schema.newMessage();
                    ProtostuffIOUtil.mergeFrom(bytes, product, schema);
                    //product 被反序列化
                    return product;
                }
            } finally {
                jedis.close();
            }

        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return null;
    }
    /**
     * 实现序列化,并存放到redis中
     *
     * @param product 要序列化的实体类
     * @return 序列化结果
     */
    public String putProduct(Product product) {
        // set Object(Product) ->序列化 ->byte[]
        try {
            Jedis jedis = jedisPool.getResource();
            try {
                String key = "product:" + product.getProductId();
                byte[] bytes = ProtostuffIOUtil.toByteArray(product, schema,
                        LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE));
                int timeout = 60 * 60; //1小时
                String result = jedis.setex(key.getBytes(), timeout, bytes);
                return result;
            } finally {
                jedis.close();
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }

        return null;
    }

}
```

##### 2、在spring-dao.xml依赖注入RedisDao
```xml
    <!--RedisDao -->
    <bean id="redisDao" class="com.redis.test.dao.cache.RedisDao">
        <!--主机ip-->
        <constructor-arg index="0" value="127.0.0.1"/>
        <!--端口号-->
        <constructor-arg index="1" value="6379"/>
        <!--timeout-->
        <constructor-arg index="2" value="10000"/>
        <!--登录redis密码-->
        <constructor-arg index="3" value="shadow123"/>
    </bean>
```

-----

#### 五、使用Junit测试RedisDao

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:spring/spring-dao.xml")
public class RedisDaoTest {

    @Autowired
    private RedisDao redisDao;

    @Autowired
    private ProductDao productDao;   //查询Product接口

    @Test
    public void testQueryProduct() {
        long id = 1000;
        Product product = redisDao.getProduct(id);  //从redis缓存中取出product
        if (null == product) {
	    //如果redis没有缓存，查询数据库，存放到redis中
            product = productDao.queryProductById(id);
            if (product != null) {
                String result = redisDao.putProduct(product);
                System.out.println("result====" +result);
                product = redisDao.getProduct(id);
                System.out.println("刚刚put的实体类==" +product);
            }
        } else {
            System.out.println("缓存===" + product);
        }

    }
}
```


-------

#### 六、Redis缓存的应用场景
在一些秒杀系统中，会有少量数据存储，高速读写访问的场景，为了降低Mysql、Oracle或SqlServer的性能消耗(如果你的系统用的是Mysql、Oracle或SqlServer数据库),可以考虑采用redis把数据在服务器端缓存。关于为什么使用Redis，可以看一下[http://www.infoq.com/cn/articles/tq-why-choose-redis/](http://www.infoq.com/cn/articles/tq-why-choose-redis/)


---------

参考文档:[http://www.cnblogs.com/549294286/p/4612601.html](http://www.cnblogs.com/549294286/p/4612601.html) <br/>
官方文档:[http://www.protostuff.io/documentation/object-graphs/](http://www.protostuff.io/documentation/object-graphs/)

项目代码:[http://git.oschina.net/zhangjiadong/RedisTest](http://git.oschina.net/zhangjiadong/RedisTest)

-------
