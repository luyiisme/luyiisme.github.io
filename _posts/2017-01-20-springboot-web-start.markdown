---
layout:     post
title:      "Spring boot web 工程启动分析"
date:       2017-01-20
author:     "luyi"
header-img: "img/post-bg-pens.jpg"
tags:
    - spring-boot
---

这里主要基于web工程分析，而非web工程相对简单点，启动过程只要启动了应用spring上下文就可以了，没有文档第2步之后的过程。

使用 Spring-boot 是可以不使用 J2EE 的标准工程开发结构的开发 web 工程的。Web 工程可以以 j2se 工程结构（包括类和资源文件）开发，运行时以工程的 main 函数入口类开始并启动 embeded Servlet 服务器。

借助[spring-boot-maven-plugin](http://docs.spring.io/spring-boot/docs/1.3.1.RELEASE/maven-plugin/examples/run-profiles.html)插件首先构建出一个uberJar.(在可执行的uberJar一文中，已经说明了repackage后的Main-Class是JarLanucher), "Start-Class"是工程定义的真正入口类（含有main函数类）。
比如，

```
@SpringBootApplication
public class SpringBootWebApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootWebApplication.class, args);
    }
}
```
## 1 .怎么判断是否web项目？

只需该应用的类加载器可以加载到下面两个类（即依赖相关jar包，spring-web,tomcat-embed-core）

```
private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };
```
是web项目那么对应的应用上下文的类就要使用"org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext"，否则是标准的j2se项目那么使用"org.springframework.context.annotation.AnnotationConfigApplicationContext"即可。两者都能通过基于注解解析bean的定义和解析的上下文，区别前者是要感知使用Servlet容器的上下文和配置。

## 2. 如何加载注解声明的bean配置

   "new BeanDefinitionLoader(registry, sources);" ,我们的入口类会作为sources之一的class对象，因此把相关配置声明基（于@Configuration，或xml)都声明或import在入口类之上，比如"@SpringBootApplication"。该注解的注解中有"@ComponentScan"注解,会把该package下一堆AutoConfiguration集合识别出来（其中包括sub package "web"下的"WebMvcAutoConfiguration",使得spring mvc可用），当然你可以使用exclude方法排除不需要的自动配置，但大部分是conditional,不满足条件是不会自动配置生效的，比如"@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)"没法加载到该类，配置就不会生效。

当我们知道其作用机制，就可以做符合项目的定制,下面以启动为例：

```
@Configuration
@Component
@ComponentScan(basePackages = {"com.alipay.sofa.xx"})
@Import(value = {
        WebMvcAutoConfiguration.EnableWebMvcConfiguration.class,
        HttpEncodingAutoConfiguration.class,
        DispatcherServletAutoConfiguration.class,
        ServerPropertiesAutoConfiguration.class,
        EmbeddedServletContainerAutoConfiguration.class})
public class Bootstrap {

    public static void main(String[] args) {
        System.setProperty(LoggingSystem.SYSTEM_PROPERTY, "org.springframework.boot.logging.log4j2.Log4J2LoggingSystem");
        SpringApplication.run(Bootstrap.class, args);

    }
}
```
## 3. 构建应用上下文过程中才启动tomcat.

这和我们传统方式不同，常规先启动容器，然后监听容器启动事件、或者在 webApp 的 webConfig 中配置特定 DispatcherServlet，亦或者基于 Servlet3 的 ServletContainerInitializer 的 SPI 来做应用层面启动逻辑。 比如 spring 针对 deployable war,是通过 spring-web.jar里默认提供的 "org.springframework.web.SpringServletContainerInitializer" 来感知容器启动并执行应用启动逻辑的。 以上的方式都是先容器启动后在应用上下文。而Spring-boot基于一个应用独占一个容器的事实，由应用上下文启动来驱动容器启动（这和传统顺序导致，一种表达是“应用部署在容器中",另外一种表达的是"应用通过借助内置容器启动来达到对外提供服务"）。Web应用的Spring应用上下文的基类是 "EmbeddedWebApplicationContext"，它在 refresh() 阶段做的就是 createEmbeddedServletContainer()行为。

> NOTE
>如果我们需要对ServletContainer做些定制，可以定义个bean 名字为"embeddedServletContainerFactory"来覆盖默认提供。
>另外在TomcatEmbeddedServletContainerFactory的初始化阶段有个EmbeddedServletContainerCustomizerBeanPostProcessor会定制(set修改属性)个ServerProperties的bean。供上下文里对Server要依赖使用的场景。

## 4. 怎么把Dispatcher登记为Servlet的呢？

"org.springframework.boot.autoconfigure.web.DispatcherServletAutoConfiguration"该自动配置中，声明所有ServletRegistration.

```
		@Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
		public ServletRegistrationBean dispatcherServletRegistration() {
			ServletRegistrationBean registration = new ServletRegistrationBean(
					dispatcherServlet(), this.server.getServletMapping());
			registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
			if (this.multipartConfig != null) {
				registration.setMultipartConfig(this.multipartConfig);
			}
			return registration;
		}
```
ServletRegistrationBean其实是个"org.springframework.boot.context.embedded.ServletContextInitializer"。（当然我们自己也可以定义多个ServletRegistrationBean那么就有了多个Servlet入口了）

那这些spring-boot定义的ServletContextInitializer执行期在哪里呢？ ：）

"org.springframework.boot.context.embedded.tomcat.TomcatStarter" 是ServletContainerInitializer。那么如果容器启动过程它可以得到执行,然后执行spring提供ServletContextInitializer就可以往servletContext中注册servlet，filter等实例了。

## 5. 静态资源如何访问

如果希望应用同时提供静态资源访问的服务，具体可以参考[官方文档](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-web-applications.html#boot-features-spring-mvc-static-content)。只要将静态资源文件（包括视图层可能使用的模板资源）都可以作为资源文件打包的Jar中。

至此，一个web项目就是以j2se的形式运行，对外提供web服务啦。


#### 著作权声明

首次发布于此，转载请保留以上链接
