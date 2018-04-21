---
layout : post
title : ajaxfileupload.js异步上传多个文件到服务器
category : springMVC
tagline: ""
date : 2016-03-06
tags : [springMVC]
---

###### 修改ajaxfileupload.js插件实现springMVC异步上传文件到服务器步骤(测试环境springMVC):

-----

###### 1、ajaxfileupload.js简介
ajaxfileupload.js是一个轻量级的插件，可轻松实现文件上传。

------

###### 2、为springMVC上传文件引入依赖
```xml
<dependency>
            <groupId>commons-fileupload</groupId>
            <artifactId>commons-fileupload</artifactId>
            <version>1.3</version>
</dependency>
```

------

###### 3、在spring配置中配置视图解析器
```xml
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver"/>
```
------------

###### 4、修改ajaxfileupload.js源码
由于ajaxfileupload.js文件只支持单个文件上传，为了实现多个文件异步上传，这里修改一下ajaxfileupload.js的源码

未修改之前的js代码

```javascript
var oldElement = jQuery('#' + fileElementId);
var newElement = jQuery(oldElement).clone();
jQuery(oldElement).attr('id', fileId);
jQuery(oldElement).before(newElement);
jQuery(oldElement).appendTo(form);
```
很容易看出，这个就是把id为什么的input加到from里去，那么要实现多个文件上传，就改成下面的样子

```javascript
if(typeof(fileElementId) == 'string'){
    fileElementId = [fileElementId];
}
for(var i in fileElementId){
    var oldElement = jQuery('#' + fileElementId[i]);
    var newElement = jQuery(oldElement).clone();
    jQuery(oldElement).attr('id', fileId);
    jQuery(oldElement).before(newElement);
    jQuery(oldElement).appendTo(form);
}
```
这样改之后，初始化的代码就要这么写：

```javascript
$.ajaxFileUpload({
    url:'/ajaxRequest',
    fileElementId:['id1','id2']//原先是fileElementId:’id’ 只能上传一个
});
```
---------------

###### 5、一个简单的前端页面demo实现连续上传2个文件
两个input框文件的name值必须一致，这样springMVC后台通过绑定参数可以直接获取文件流

```
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title></title>
    <script src="../js/jquery-1.11.3.min.js"></script>
    <script src="../js/ajaxfileupload.js"></script>
    <script type="text/javascript">
        function ajaxFileUpload(){
            var uname = "shadow";
            //开始上传文件时显示一个图片,文件上传完成将图片隐藏
            //$("#loading").ajaxStart(function(){$(this).show();}).ajaxComplete(function(){$(this).hide();});
            //执行上传文件操作的函数
            $.ajaxFileUpload({
                //处理文件上传操作的服务器端地址(可以传参数,已亲测可用)
                url:'/test/fileUpload?uname=' + uname,
                secureuri:false,                       //是否启用安全提交,默认为false
                fileElementId:['myBlogImage1','myBlogImage2'],           //文件选择框的id属性
                dataType:'text',                       //服务器返回的格式,可以是json或xml等
                success:function(data, status){        //服务器响应成功时的处理函数
                    $('#result').html('修改头像成功' + data);
                },
                error:function(data, status, e){ //服务器响应失败时的处理函数
                    $('#result').html('图片上传失败，请重试！！');
                }
            });
        }
    </script>
</head>
<body>
<div id="result"></div>
<input type="file" id="myBlogImage1" name="myfiles"/>
<input type="file" id="myBlogImage2" name="myfiles"/>
<input type="button" value="上传图片" onclick="ajaxFileUpload()"/>
</body>
</html>
```
-----------

###### 6、后台获取前端异步传入的2个文件

```java
@Controller
@RequestMapping("/test")
public class FileUploadController {

    @RequestMapping(value = "/fileUpload")
    public String addUser(@RequestParam("uname") String uname, @RequestParam MultipartFile[] myfiles,
                          HttpServletRequest request, HttpServletResponse response,
                          String birth) throws IOException {
        response.setContentType("text/plain; charset=UTF-8");
        //可以在上传文件的同时接收其它参数
        uname = new String(uname.getBytes("iso-8859-1"),"utf8");
        System.out.println("收到用户[" + uname + "]的文件上传请求");
        //如果用的是Tomcat服务器，则文件会上传到\\%TOMCAT_HOME%\\webapps\\YourWebProject\\upload\\文件夹中
        //这里实现文件上传操作用的是commons.io.FileUtils类,它会自动判断/upload是否存在,不存在会自动创建
      /*  String realPath = request.getSession().getServletContext().getRealPath("/upload");*/
        //设置响应给前台内容的数据格式

        //设置响应给前台内容的PrintWriter对象
        PrintWriter out = response.getWriter();
        //上传文件的原名(即上传前的文件名字)
        String originalFilename = null;
        //如果只是上传一个文件,则只需要MultipartFile类型接收文件即可,而且无需显式指定@RequestParam注解
        //如果想上传多个文件,那么这里就要用MultipartFile[]类型来接收文件,并且要指定@RequestParam注解
        //上传多个文件时,前台表单中的所有<input type="file"/>的name都应该是myfiles,否则参数里的myfiles无法获取到所有上传的文件
        for (MultipartFile myfile : myfiles) {
            if (myfile.isEmpty()) {
                out.print("1`请选择文件后上传");
                /*out.flush();*/
                return null;
            } else {

                originalFilename = myfile.getOriginalFilename();
                System.out.println(originalFilename);
                //得到tmp的绝对路径
                String tmpPath = request.getServletContext().getRealPath("views") + "\\";
                //System.out.println("根路径" + request.getServletContext().getRealPath("tmp") );
                String resetName = Long.toString(System.currentTimeMillis());
                String testPath = tmpPath + resetName +".jpg";
                myfile.transferTo(new File(testPath));
                System.out.println("文件原名: " + originalFilename);
                System.out.println("文件名称: " + myfile.getName());
                System.out.println("文件长度: " + myfile.getSize());
                System.out.println("文件类型: " + myfile.getContentType());
                System.out.println("========================================");
            }
        }
        out.write("success");

        return null;
    }
}

```
这样就实现ajaxfileupload.js多文件上传
------------

###### 7、[demo项目源码](http://git.oschina.net/zhangjiadong/Ajaxfileupload)

---------
**你是你梦想路上唯一的高墙，越过去，全世界都能看到你的光亮。**
-----------<small>夜空中最亮的星</small>

------------
