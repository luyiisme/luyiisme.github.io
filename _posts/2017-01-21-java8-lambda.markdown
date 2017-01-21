---
layout:     post
title:      "Java8 Lambda使用与原理"
date:       2017-01-21
author:     "luyi"
header-img: "img/post-bg-pens.jpg"
tags:
    - Java8
    - Lambda
---

## 1.Lambda相关概念与特性

Lambda 表达式是一个匿名函数，源于数学λ演算。是闭包函数，但闭包并不一定是Lambda 函数。
它可以赋值给变量，作为函数参数，作为函数返回值。

## 2.在Java8中Lambda的语法

```
        List<Integer> integers = Arrays.asList(2, 4, 6, 8);

        //老的方式
        for (Integer x : integers) {
            System.out.println(x);
        }

        //1.8 非lambda
        integers.forEach(new Consumer<Integer>() {
            @Override
            public void accept(Integer x) {
                System.out.print(x);
            }
        });

        integers.forEach((x) -> System.out.println(x));

        integers.forEach(x -> System.out.println(x));

        //可以赘述其参数类型
        integers.forEach((Integer x) -> System.out.println(x));


        //多行实现
        integers.forEach((x) -> {
            x = x * 10;
            System.out.println(x);
        });

        // 本地变量
        integers.forEach((x) -> {
            int y = x + 10;
            System.out.println(y);
        });
```

### Lambda 表达式的生命周期（Java 8）

那么Java 8是如何处理和支持Lambda表达式的呢？

 - 编译器在类中生成一个静态函数；
 - 运行时调用该静态函数；（invokeStatic?）

```
x -> System.out.println(x);
```
编译为:

```
public static void generatedNameOfLambadaFunction(Integer x){
 System.out.println(x)
}
```

举个简单的例子，代码例子如下：

```
public class LifeCycleExample {
    public static void main(String[] args) {
        List<Integer> integers = Arrays.asList(2, 4, 6, 8);
        integers.forEach(x -> System.out.println(x));
    }
}
```
先编译为字节码，然后用反汇编工具对class文件执行“javap -private”显示private可见性以上的主要的类和成员，

```
  java8demo git:(master)  javap -private  target/classes/org/luyi/lambda/syntax/LifeCycleExample.class

Compiled from "LifeCycleExample.java"
public class org.luyi.lambda.syntax.LifeCycleExample {
  public org.luyi.lambda.syntax.LifeCycleExample();
  public static void main(java.lang.String[]);
  private static void lambda$main$6(java.lang.Integer);
}
```
除了构造函数和 main 函数，多了一个对应 Lambda 表达式的私有静态方法，最后该方法会被调用执行。


但是如果编译后是“invokestatic”虚拟机命令，返回类型又是void,那么Lambda 表达式是什么类型呢？后面会继续说明，这里我们可以先看下实际的字节码，

```
39: invokedynamic #5,  0              // InvokeDynamic #0:accept:()Ljava/util/function/Consumer;
```
invokeDynamic( 从JDK7 开始提供的，为了支持动态类型语言在运行时才能确定接收者的类型的场景)方法调用，运行时首次解析，生成一个匿名内部类。可以dump内存看看到该类

![image](/img/in-post/lambda-inner-class.png)

由"java.lang.invoke.LambdaMetafactory"的静态方法生成

```
public static CallSite metafactory(MethodHandles.Lookup caller,
                                       String invokedName,
                                       MethodType invokedType,
                                       MethodType samMethodType,
                                       MethodHandle implMethod,
                                       MethodType instantiatedMethodType)
```

***那一个Lambda表达式是什么类型呢？primitve,Object?***

许多语言有专门的函数（数据）类型，但是Java8 处于向前兼容，和避免类型复杂化等多种原因考虑没有引入新类型；


## 3.Functional Interface 函数式接口

Functional Interface是***只提供一个方法***的普通接口。

```
public interface Consumer<T> {
    void accept(T t);
...

}
```
在 Java8 前，JDK 已经提供哪些函数式接口了，

```
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}

java.lang.Runnable,
java.lang.Comparable,
java.util.concurrent.Callable;
```
JDK8 又新增了哪些通用的？你知道他们各自的使用场景吗？

```
java.util.function.Consumer<T>,
java.util.function.Supplier<T>,
java.util.function.Predicate<T>,
java.util.function.Function<T,R>,
```

为什么需要函数式接口？

它表达了 Lambda 表达式的类型，函数式接口是方法签名（signature），lambda表达式是方法body,两者组成了一个整体。

### 可以将 Lambda 表达式赋值给函数接口的局部变量

```
      Consumer<Integer> consumer=x -> System.out.println(x);
        integers.forEach(consumer);
```

注解 @FuncationalInterface 推荐使用(考虑向前兼容又非必须)，Java8 编译器会帮你保证；

```
    lambda git:(master)  javac Echo.java
syntax/Echo.java:6: error: Unexpected @FunctionalInterface annotation
@FunctionalInterface
^
  Echo is not a functional interface
    multiple non-overriding abstract methods found in interface Echo
1 error
```

所以lambda 表达式的类型是函数式接口类型，前面不是说lambda 是个静态函数，为啥能赋值给个函数式接口；

那么是不是lambda的实现是基于匿名内部类的形式?No(但本质来说可以说 YES);

## 4.变量捕捉（Capture 本地变量,静态类属性，实例属性）

```
   List<Integer> integers = Arrays.asList(2, 4, 6, 8);
        int var=1;
        integers.forEach(x -> System.out.println(x+var));
```
会有编译异常吗？

实际上,如果JDK8前的编译器下的匿名内部类会编译不通过；

```
       int var=0;
        Runnable r=new Runnable() {
            @Override
            public void run() {
                System.out.print(var+2);
            }
        };

javac 1.7.0_79
  lambda git:(master)  javac CaptureExample.java
CaptureExample.java:22: error: local variable var is accessed from within inner class; needs to be declared final
                System.out.print(var+2);
                                 ^
```
但是使用 JDK8的编译器是通过的。声明final是个推荐的，但即使不声明，若实际效果是final也行。

如果我们尝试改变其值呢？

```
        int var=1;
        integers.forEach(x -> {
            var++;
            System.out.println(x+var);});
    }
```
会得到编译异常,这和匿名内部类情况下约束是类似的，
```
  lambda git:(master)  javac CaptureExample.java
CaptureExample.java:15: error: local variables referenced from a lambda expression must be final or effectively final
            var++;
            ^
```
Lambda如何处理捕捉到的变量的呢？反编译后，我们发现静态的lambda函数增加了一个对应类型的参数；

```
  private static void lambda$main$6(int, java.lang.Integer);
```

### 关于Lambda 表达式内的this指针； 指向谁呢？

首先看先静态方法中使用this指针，

```
    public static void main(String[] args) {
        integers.forEach(x -> {
            System.out.println(this.toString());
            System.out.println(x);
        });

        integers.forEach(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) {
                System.out.println(this.toString());
            }
        });
   }
```
前者会有编译异常,

```
ThisPointerExample.java:15: error: non-static variable this cannot be referenced from a static context
            System.out.println(this.toString());
```

而匿名内部类形式和以前JDK保持兼容，表示的是Consumer这个new的实例“org.luyi.lambda.ThisPointerExample$1@3cd1a2f1”。

现在讲整数遍历方法作为一个实例方法，然后再调用

```
public class ThisPointerExample {
    public void doSth() {
        List<Integer> integers = Arrays.asList(2, 4, 6, 8);
        integers.forEach(x -> {
            System.out.println(this.toString());
            System.out.println(x);
        });

    }
    public static void main(String[] args) {
        new ThisPointerExample().doSth();
    }
}
```
执行结果可以发现，Lambda 表达式内部的this指针是指向其 enclosing Class;


***那么，Lambda 表达 与 匿名内部类的区别可以总结了下了***：

 - 后者可以有实例的属性变量（状态）；
 - 后者可以含有多个方法；
 - 方法体内的this指针的指向；

## 5.方法引用

Lambda 虽然很方便构造个匿名功能函数，但是有些功能函数的实现已经存在，还要重新在写个Lambda或者去使用对应的方法？ 方法引用提供了便捷的处理方式；

### 引用一个静态方法；

```
        integers.forEach(x -> {
           // old style
           //System.out.println(String.valueOf(x));

            Function<Integer, String> i2s = String::valueOf;
            System.out.println(i2s.apply(x));
        });
```
要求：被引用方法的签名需要与函数接口签名匹配；

### 引用一个构造函数；

```
//        System.out.println(new Integer("11"));

        Function<String,Integer> s2i=Integer::new;
        System.out.println(s2i.apply("11"));
```

### 引用一个实例方法；

```
 Consumer<Object> sysout = System.out::println;
        sysout.accept("hello world");
```


## 6.默认方法（Default Method）

问题背景，接口进化演进的问题，修改任何一个接口，比如增加个接口方法，那么所有以前接口的实现都要做响应的调整，破坏了原有稳定。 比如JDK8希望在集合类型（List,Set...）中增加新遍历方法“forEach(Consumer<? super T> action)” =>默认方法。

```
public interface Iterable<T> {

    Iterator<T> iterator();

    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }

    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
```
静态方法，有实现并且能够被接口实现类所继承；

### 默认方法被继承

```
public interface Test {

    default void doSth(){
        System.out.println("hello");
    }
}

public class DefaultMethodExample implements Test {

    public static void main(String[] args) {
        DefaultMethodExample impl = new DefaultMethodExample();
        impl.doSth();//hello
    }
}
```

### 覆写默认方法

```
public class DefaultMethodExample implements Test {

    @Override
    public void doSth() {
        System.out.println("hi");
    }

    public static void main(String[] args) {
        DefaultMethodExample impl = new DefaultMethodExample();
        impl.doSth();//hi
    }
}
```

### 多级继承覆盖

```
public interface Test2 extends Test {
    @Override
    default void doSth() {
        System.out.println("hello2");
    }
}

public class DefaultMethodExample implements Test2 {
    public static void main(String[] args) {
        DefaultMethodExample impl = new DefaultMethodExample();
        impl.doSth();//hello2
    }
}
```

### 冲突

```
public interface A {
    default void doSth(){
        System.out.println("a");
    }
}
public class ConflictionExample implements Test,A {
}
```
接口A也提供了doSth的默认方法，如果一个实现类同时实现Test,A两个接口，那么编译器会报错误
```
Error:(6, 8) java: 类 org.luyi.lambda.defmtd.ConflictionExample从类型 org.luyi.lambda.defmtd.Test 和 org.luyi.lambda.defmtd.A 中继承了doSth() 的不相关默认值
```
解决冲突的方式是，实现类中重新覆写doSth方法,可以重新写实现，也可以使用任务一个父实现；

![image](https://luyiisme.github.io/img/in-post/lambda-override.png)

---

#### 著作权声明

`首次发布于此，转载请保留以上链接`
