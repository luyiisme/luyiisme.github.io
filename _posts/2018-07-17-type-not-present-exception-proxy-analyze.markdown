---
layout:     post
title:      "java.lang.ArrayStoreException: sun.reflect.annotation.TypeNotPresentExceptionProxy的错误分析"
date:       2018-07-17
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - ArrayStoreException
    - SpringBoot
---

笔者使用 springboot， 在启动中遇到过 “Caused by: java.lang.ArrayStoreException: sun.reflect.annotation.TypeNotPresentExceptionProxy” 异常，查看了一些网文都没有看到比较准确的分析。经过自己的排查将问题的原因和排查方法总结于此。导致该问题的原因形式可能多种多样，但原理和分析方法相对有章可循的。

这个异常一般是由于某些地方尝试反射分析某个类文件的注解元数据时导致的失败。那么是在分析哪个类文件的注解呢？先定位找到该类文件。

下面是举例某个场景（spring mvc 分析哪些 bean 是 controller 的逻辑）时抛出该错误，打印出对应的错误栈看到：

```
...
Caused by: java.lang.ArrayStoreException: sun.reflect.annotation.TypeNotPresentExceptionProxy
	at sun.reflect.annotation.AnnotationParser.parseClassArray(AnnotationParser.java:724)
	at sun.reflect.annotation.AnnotationParser.parseArray(AnnotationParser.java:531)
	at sun.reflect.annotation.AnnotationParser.parseMemberValue(AnnotationParser.java:355)
	at sun.reflect.annotation.AnnotationParser.parseAnnotation2(AnnotationParser.java:286)
	at sun.reflect.annotation.AnnotationParser.parseAnnotations2(AnnotationParser.java:120)
	at sun.reflect.annotation.AnnotationParser.parseAnnotations(AnnotationParser.java:72)
	at java.lang.Class.createAnnotationData(Class.java:3521)
	at java.lang.Class.annotationData(Class.java:3510)
	at java.lang.Class.createAnnotationData(Class.java:3526)
	at java.lang.Class.annotationData(Class.java:3510)
	at java.lang.Class.getAnnotation(Class.java:3415)
	at java.lang.reflect.AnnotatedElement.isAnnotationPresent(AnnotatedElement.java:258)
	at java.lang.Class.isAnnotationPresent(Class.java:3425)
	at org.springframework.core.annotation.AnnotatedElementUtils.hasAnnotation(AnnotatedElementUtils.java:573)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping.isHandler(RequestMappingHandlerMapping.java:177)
  ...
```

然后选择一个合适的方法帧进行 DEBUG 分析究竟是在处理到哪个 class 时，随后就异常退出了。
JDK 的工具 package 太低阶了，所以这里可以在 spring 的 “RequestMappingHandlerMapping.isHandler”方法的 177 行进行断点。

- 断点会发现疑似问题类的实在太多了看得有点累，那建议条件断点！ 一般出错的不会spring的基础jars，而是我们自己写的类或者是写二方包的类，可以通过条件断点进行适当的报名过滤；

- 或者使用 btrace，打印出访问该方法行的每次入参“类”的名称。最后一次打印出来的就是你要找的“家伙”了，因为分析它时接着就抛异常，退出主程序了。

分析该问题类文件的注解，你会发现哪个注解有问题了。尤其注意看下这些注解里哪些引用了外部Class。比如:

```
@Configuration
...
@AutoConfigureBefore({ Xx.class })
public class Yyyy {

```

比较明显就定位怀疑到 “Xx.class”,且它不在 classpath 中。

补充：一旦定位到是哪个类文件后，如果在 Intellj idea 中打开该类文件可以明显看到注解方法的引用的一些类是“红色”（不存在 classpath中）更显著。
