---
layout : post
title : 简单工厂模式
category : 设计模式
tagline: ""
date : 2016-06-25
tags : [设计模式]
---

### 1、模式场景
有这样一个简单的软件应用场景，开发一个简单的计算器，对两个数进行加减乘除，可以对两个数提供不同的
加减乘除运算符按钮，如果我们在使用这些按钮时，不需要知道这些按钮具体的对象，只需要知道表示该按钮
的参数，并提供一个调用方便的方法，把该参数传入方法即可返回对应的运算符按钮对象，此时，我们可以使
用简单工厂模式。



### 2、 模式定义
**简单工厂模式(Simple Factory Pattern)**: 又称为静态工厂方法(Static Factory Method)模
式,他属于创建型模式。在简单工厂模式中，它可以根据参数的不同返回不同类的实例。简单工厂模式专门定义
一个类来负责创建具体类的实例，被创建的类通常都具有相同的父类。


### 3、 模式结构
简单工厂模式包含如下角色:

**Factory**: 工厂角色负责实现创建所有实例的内部逻辑

**Product**: 抽象产品角色是所创建的所有对象的父类，负责描述所有实例所共有的公共接口

**ConcreteProduct**: 具体产品角色 ,都继承产品角色，是每一个具体类的实现


### 4、具体代码实现
1)、创建Product抽象产品角色:对两个数对象operator,getResult()返回对象的结果。

```java
public class Operation {
  protected double a = 0;
  protected double b = 0;
  
  public double getResult() {
    return 0.0;
  }
  
  //省略get set方法
}
```

2)、ConcreteProduct具体产品创建

```java
/** 
 加法
*/
public class OperationAdd extends Operation {
  
  @Override
  public double getResult() {
    return a + b;
  }
}
/**
 减法
*/
public class OperationSub extends Operation {
  
  @Override
  public double getResult() {
    return a - b;
  }
}
/**
 乘法
*/
public class OperationMul extends Operation {
  
  @Override
  public double getResult() {
    return a * b;
  }
}
/**
 除法
*/
public class OperationDiv extends Operation {
  
  @Override
  public double getResult() {
    return a / b;
  }
}
```

3)、最重要的Factory工厂创建

```java
public class OperationFactory {
  
  public static Operation creareOperation(String operation) {
    Operation oper = null;
    switch (operation) {
      case "+": {
        oper = new OperationAdd();
        break;
      }
      case "-": {
        oper = new OperationSub();
        break;
      }
      case "*": {
        oper = new OperationMul();
        break;
      }
      case "/": {
        oper = new OperationDiv();
        break;
      }
    }
    return oper;
  }
}
```

4)、测试

```java
public class TestOperation {
  public static void main(String[] args) {
    Operation operation = OperationFactory.creareOperation("-");
    operation.setA(10);
    operation.setB(20);
    System.out.println("result==" + operation.getResult());
  }
}
```

### 5、分析
通过测试类可以看出，**简单工厂模式(Simple Factory Pattern)**当你需要什么，只需要传入对应参数，
就可以获取你需要的对象，无需知道其具体类的创建细节。


### 6、模式优缺点
**优点:**  将对象的创建和对象本身业务处理分离，降低系统的耦合度，使其两者修改变的比较容易。对于创建具体实例的过程全部由
工厂角色承担。

**缺点:** Factory角色的职责过于太重，增加新的运算操作实体类需要进行修改其判断逻辑。如果具体的运算操作太多可能工厂类维护
太过麻烦。

### 7、[demo源码](https://github.com/zhangjiadong/DesignPattern)

