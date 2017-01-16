---
layout:     post
title:      "可执行的uberJar (fatJar)"
date:       2017-01-16
author:     "luyi"
header-img: "img/post-bg-js-version.jpg"
tags:
    - spring boot ,fatJar
    - uberJar
---

将2016-01月的文章整理了下，就当前常用的可执行Jar（Executable Jar,Runnable Jar）的实现方案做下总结.

JAR，一个jar文件是java 的jar工具构建的zip压缩文件，里面包含类文件，资源文件； 可以通过jar (或者zip)进行解压缩。

举个栗子，通过 maven 工程原型创建个j2se工程(工程名：uberJar ，什么是uberJar后面会讲)：

```
uberjar
├── pom.xml
└── src
    ├── main
    │   └── java
    │       └── org
    │           └── luyi
    │               └── App.java
    └── test
        └── java
            └── org
                └── luyi
                    └── AppTest.java
```

看下其jar结构

```
 > target  jar -tf uberjar-1.0-SNAPSHOT.jar
...
META-INF/MANIFEST.MF
...
org/luyi/App.class
...
```

## 1. 什么叫可执行jar?

只需要在jar文件的MF文件中定义" Main-Class: myPrograms.MyClass",声明个入口类，通过"java -jar xx.jar"即能执行。这个是再熟悉不过的基本常识，但还是啰嗦下，顺便提下如何借助maven的jar插件的编辑MF的方式。

```
>  target  java -jar uberjar-1.0-SNAPSHOT.jar
uberjar-1.0-SNAPSHOT.jar中没有主清单属性
```
说明，uberJar项目默认打成的jar不是可执行jar,需要对MF进行定义，显示配置 maven-jar-plugin

```
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>2.6</version>
        <configuration>
          <archive>
            <manifest>
              <mainClass>org.luyi.App</mainClass>
            </manifest>
          </archive>
        </configuration>
      </plugin>
    </plugins>
  </build>
```
现在jar中MF文件，最后一行就多了入口类，

```
>  target  cat  META-INF/MANIFEST.MF
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: luyi
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_25
Main-Class: org.luyi.App
```
再执行，

```
>  target  java -jar uberjar-1.0-SNAPSHOT.jar
Hello World!
```
ok,那么一个可执行的jar制作完成，当然以上最基本可执行jar。

## 2. 那什么是可执行的uberJar，什么是uberJar？

同样继续刚才的例子,我们现在uberJar的pom.xml中依赖gson,并且 org.luyi.App.class 不在简单打印"Hello World!",而是改为以 json 格式打印出虚拟机的环境变量

```
        System.out.println(new Gson().toJson(System.getenv()));

```
执行打包`java -jar uberjar-1.0-SNAPSHOT.jar`就会报类无法找到异常`Exception in thread "main" java.lang.NoClassDefFoundError: com/google/gson/Gson`。因为gson jar 包在哪里是不知道的。通过在MF中添加`Class-Path`配置，指定向gson jar文件位置即可[详细参考](http://stackoverflow.com/questions/1510071/maven-how-can-i-add-an-arbitrary-classpath-entry-to-a-jar)(另外注意：java "-cp,-jar"不能同时使用。)。 那么userjar.jar分发到哪里，gson.jar也要放在MF中指定位置才行。能不能直接拿到，执行uberJar.jar就行了，不需要再关心维护它的依赖呢？那么就需要将所有的依赖都一同放在uberJar.jar里。

### 2.1  uberJar(fatJar)与运行

uber 是德语 ，"over"的含义。也叫fatJar,比起普通jar，fatJar将所有的该原生jar的依赖都包含在内。

下面介绍下基于Maven工具怎么构建fatJar，和各自区别。

####  「1」 Apache 官方提供的[maven-shadow-plugin](http://maven.apache.org/plugins/maven-shade-plugin/)

```
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.2</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```
这插件会把依赖的jar的文件覆盖到原生jar文件里，也就是其所有的依赖jar都被解压且合并进来了

```
>  target  jar -tf uberjar-1.0-SNAPSHOT.jar
...
META-INF/MANIFEST.MF
...
org/luyi/App.class
...
com/google/gson/annotations/Expose.class
...
```
这种形式比较简单，但会有一些问题需要注意，

（1）万一uberjar.jar被其他应用依赖，其他系统也依赖gson而且版本不一致容易运行时出现冲突，解决方法[class-relocation](http://maven.apache.org/plugins/maven-shade-plugin/examples/class-relocation.html)
（2）如果uberjar.jar的依赖里有同名的资源文件怎么办简单覆盖还是追求，解决方法[Resource Transformers](http://maven.apache.org/plugins/maven-shade-plugin/examples/resource-transformers.html)。比如spring的xml解析资源文件

```
<configuration>
              <transformers>
                <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                  <resource>META-INF/spring.handlers</resource>
                </transformer>
                <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                  <resource>META-INF/spring.schemas</resource>
                </transformer>
              </transformers>
            </configuration>
```


####  「2」第三方插件[onejar-maven-plugin](https://onejar-maven-plugin.googlecode.com/svn/mavensite/usage.html)

配置插件，

```
<plugin>
		<groupId>com.jolira</groupId>
		<artifactId>onejar-maven-plugin</artifactId>
		<version>1.4.4</version>
		<executions>
		  <execution>
			<goals>
				<goal>one-jar</goal>
			</goals>
		  </execution>
		</executions>
	</plugin>
```
打包后，出现" uberjar-1.0-SNAPSHOT.one-jar.jar",

```
>  target  jar -tf uberjar-1.0-SNAPSHOT.one-jar.jar
META-INF/MANIFEST.MF
main/uberjar-1.0-SNAPSHOT.jar
lib/gson-2.5.jar
...
src/OneJar.java
src/com/simontuffs/onejar/Boot.java
src/com/simontuffs/onejar/Handler.java
src/com/simontuffs/onejar/IProperties.java
src/com/simontuffs/onejar/JarClassLoader.java
src/com/simontuffs/onejar/OneJarFile.java
src/com/simontuffs/onejar/OneJarURLConnection.java
```
多了很多新生成的类，再来看看MANIFEST.MF,

```
> cat META-INF/MANIFEST.MF
Manifest-Version: 1.0
ImplementationVersion: 1.0-SNAPSHOT
Main-Class: com.simontuffs.onejar.Boot
```
是的，`Main-Class`已经不是我们自己定义的'org.luyi.App'了。gson jar文件也以 "jar in jar"形式存在着"lib/gson-2.5.jar"。那么这个Boot入口做了那些事呢？

（1） 确定当前one-jar所在jar文件位置。 这是分析jar里内容的前提。方法是从classpath中分析出该Boot类所在的jar;
（2）构建新的classLoader即JarClassLoader,基于"main/uberjar-1.0-SNAPSHOT.jar","lib/gson-2.5.jar";
（3）找到原生fatjar --"main/uberjar-1.0-SNAPSHOT.jar"的实际定义的main函数类，并通过jarClassLoader加载之，反射调用之；

#### 「3」最后介绍下Spring-boot项目提供的[Spring Boot Maven Plugin](http://docs.spring.io/spring-boot/docs/1.3.1.RELEASE/maven-plugin/examples/run-profiles.html)。

 "org.springframework.boot：spring-boot-maven-plugin"的maven plugin，同样解决了uberJar的运行问题，处理逻辑与 onejar-maven-plugin 差不多。
引入，

```
         <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>1.3.1.RELEASE</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```
执行打包命令后，target目录

```
...
-rw-r--r--  1 luyi  staff   313K  1 18 16:17 uberjar-1.0-SNAPSHOT.jar
-rw-r--r--  1 luyi  staff   2.6K  1 18 16:17 uberjar-1.0-SNAPSHOT.jar.original
```
".original"标识的是原生的jar,而"uberjar-1.0-SNAPSHOT.jar"是个repackage执行出的uberJar,可以看下其结构

```
>  target  jar -tf uberjar-1.0-SNAPSHOT.jar
...
META-INF/MANIFEST.MF
...
org/luyi/App.class
...
lib/gson-2.5.jar
...
org/springframework/boot/loader/JarLauncher.class
```

在MF新增的Main-Class 是:
>Main-Class: org.springframework.boot.loader.JarLauncher

同时原MainClass也直接声明在了文件信息了：
>Start-Class: org.luyi.App

而"org/luyi/App.class"在此jar里， 这与one-jar不一样的是，没有jar原生jar整体放在main目录。

该spring-boot的maven插件，有个比较好的设计是，借助个 java.lang.Runnable 的实现类 `org.springframework.boot.loader.MainMethodRunner`, 用全新线程，持有全新的上下文类加载器运行该应用，然后现有的启动线程完成这一系列处理工作就结束其生命。

因此，通过One-Jar,Spring-boot的Maven插件相对方便的打包出一个可运行的uberJar。部署起来是不是方便快捷。


#### 著作权声明

首次发布于此，转载请保留以上链接
