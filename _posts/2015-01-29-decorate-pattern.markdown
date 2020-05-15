---
layout:     post
title:      "装饰者模式"
subtitle:   ""
date:       2016-05-06
author:     "Shadow"
header-img: "post-bg-2015.jpg"
tags:
    - 设计模式
---

**一、装饰者模式**，是面向对象编程领域中，一种动态地往一个类中添加新的行为的设计模式。就功能而言，修饰模式相比生成子类更为灵活，这样可以给某个对象而不是整个类添加一些功能。
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过使用修饰模式，可以在运行时扩充一个类的功能。原理是：增加一个修饰类包裹原来的类，包裹的方式一般是通过在将原来的对象作为修饰类的构造函数的参数。装饰类实现新的功能，但是，在不需要用到新功能的地方，它可以直接调用原来的类中的方法。修饰类必须和原来的类有相同的接口。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;修饰模式是类继承的另外一种选择。类继承在编译时候增加行为，而装饰模式是在运行时增加行为。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当有几个相互独立的功能需要扩充时，这个区别就变得很重要。在有些面向对象的编程语言中，类不能在运行时被创建，通常在设计的时候也不能预测到有哪几种功能组合。这就意味着要为每一种组合创建一个新类。相反，修饰模式是面向运行时候的对象实例的,这样就可以在运行时根据需要进行组合。一个修饰模式的示例是JAVA里的`Java I/O Streams`的实现。

------

**二、装饰者模式UML结构图**

![Java And Unicode](/images/decorated.jpg)


**Component**:抽象构件。是定义一个对象接口，可以给这些对象动态地添加职责。
**ConcreteComponent**:具体构件。是定义了一个具体的对象，也可以给这个对象添加一些职责
**Decorator**:抽象装饰类。是装饰抽象类，继承了Component,从外类来扩展Component类的功能，但对于Component来说，是无需知道Decorator存在的。
**ConcreteDecorator**:具体装饰类，起到给Component添加职责的功能。

------

**三、实现装饰者模式**
例如:一个窗口系统中的窗口，允许这个窗口内容滚动，我们希望给它添加水平或垂直滚动条。假设窗口通过“Window”类实例来表示，并且假设它没有添加滚动条功能。

java code实现

```java
//window接口
public interface Window {
    void draw();

    String getDescription();
}
```
---------
//SimpleWindow是一个实现了Windows的简单窗体

```java
public class SimpleWindow implements Window {
    @Override
    public void draw() {
        //绘制窗体
    }

    @Override
    public String getDescription() {
        return "简单的窗体";
    }
}
```

--------
WindowDecorator修饰抽象类,对Window没有人任何影响

```java
abstract class WindowDecorator implements Window {

    protected Window decorateWindow;

    public WindowDecorator(Window decorateWindow){
        this.decorateWindow = decorateWindow;
    }
}
```
------
VerticalScrollBarDecorator垂直滚动装饰实现

```java
public class VerticalScrollBarDecorator extends WindowDecorator {

    public VerticalScrollBarDecorator(Window decorateWindow) {
        super(decorateWindow);
    }

    @Override
    public void draw() {
        drawVerticalScrollBar();
        decorateWindow.draw();
    }

    private void drawVerticalScrollBar() {
        //添加垂直滚动条
    }

    @Override
    public String getDescription() {
        return decorateWindow.getDescription() + "包含垂直滚动条";
    }


}
```
-----------

HorizontalScrollBarDecorator 水平滚动装饰实现

```java
public class HorizontalScrollBarDecorator extends WindowDecorator {

    public HorizontalScrollBarDecorator(Window decoratedWindow) {
        super(decoratedWindow);
    }

    @Override
    public void draw() {
        drawHorizontalScrollBar();
        decorateWindow.draw();
    }

    private void drawHorizontalScrollBar() {
        //绘制水平滚动条
    }

    @Override
    public String getDescription() {
        return decorateWindow.getDescription() + "包含水平滚动条";
    }
}
```
---------
DecoratedWindowTest测试类，成功扩展了window的功能，添加了水平和垂直滚动的功能

```java
public class DecoratedWindowTest {
    public static void main(String[] args) {
        //简单窗体
        Window simpleWindow = new SimpleWindow();
        System.out.println(simpleWindow.getDescription());
        //添加水平滚动条修饰
        Window decoratedWindow1 = new HorizontalScrollBarDecorator(new SimpleWindow());
        System.out.println(decoratedWindow1.getDescription());
        //添加垂直滚动条修饰
        Window decorateWindow2 = new VerticalScrollBarDecorator(new SimpleWindow());
        System.out.println(decorateWindow2.getDescription());
    }
}

```