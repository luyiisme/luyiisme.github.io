---
layout:     post
title:      "JAX-RS REST 服务结果的自动封装"
date:       2017-06-23
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - REST
---

当使用遵循 JAX-RS 标准的框架开发REST 服务时，我们倾向于定义个（含有JAX-RS）注解接口。 服务器端负责实现该接口，而客户端是该接口的代理进行远程调用，从而简化开发。比如，下面是个具体的 REST API 接口的例子:

```
@Path("/movies")
public interface ServicesInterface {

    @GET
    @Path("/getinfo")
    @Produces({ MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML })
    Movie movieByImdbId(@QueryParam("imdbId") String imdbId);
```

Movie 对象是个典型的资源。

但是如果客户端不支持JAX-RS(比如，不是JAVA语言)，尤其浏览器端请求的情况下，一般都需要有固定的JSON/XML的数据结构，比如,响应体的数据结构:

```
//失败
{"error":{"message":"xxxx","code":500},"success":false}
//成功
{"data":{"name":"jack"},"success":true}
```

但这个需求，就会使得我们给方法返回类型加个封装类:

```
...
  Result<Movie> movieByImdbId(@QueryParam("imdbId") String imdbId);
```

这样可以满足需求，但如果你不希望封装类型侵入的到接口方法中呢（处于简洁或者不想做“打包”工作）？ 下面提供一个解决方案。

## 通过JAX-RS 客户端访问结果无封装，其他客户端访问默认对结果进行封装；

- 首先，在服务器端（服务实现端）声明一个加封装类型的拦截器:

```
@Provider
public class WrapResultJaxrsFilter implements ContainerRequestFilter, WriterInterceptor, ContainerResponseFilter {

    @Override
    public void filter(ContainerRequestContext requestContext) throws IOException {
        String wrValue = requestContext.getHeaderString("Not-Wrap-Result");
        if (StringUtils.isEmpty(wrValue)) {
            //need wrap result
            requestContext.setProperty("wrap-result", "T");
        }
    }

    @Override
    public void aroundWriteTo(WriterInterceptorContext context) throws IOException, WebApplicationException {
        //省略 @ResultWrapper 的自定义注解处理逻辑；

        Object originalObj = context.getEntity();
        if (context.getProperty("wrap-result") != null && !(originalObj instanceof Result)) {
            context.setEntity(Result.ok().data(originalObj));
            context.setType(Result.ok().getClass());
        }
        context.proceed();
    }

    /**
     * wrapper http status 204 to 404
     */
    @Override
    public void filter(ContainerRequestContext requestContext, ContainerResponseContext responseContext) throws IOException {
      if (requestContext.getProperty("wrap-result") != null&&responseContext.getStatus() == 204 && !responseContext.hasEntity()) {
            responseContext.setStatus(404);
            responseContext.setEntity(Result.error().code(404).message("Resource Not Found!"));
            responseContext.getHeaders().add(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON);
        }
    }
}
```

第一个方法是实现 “ContainerRequestFilter” 接口，用来判断请求头里是否暗示不要对响应结果进行封装；

第二个方法是实现 “WriterInterceptor”接口，针对需要封装的请求对结构进行封装处理。这里需要注意的是对返回类型已经是封装类（比如：异常处理器的响应可能已经是封装类型）时要忽略掉。

第三个方法是实现“ContainerResponseFilter”，它的目的是专门处理方法返回类型是 void,或者某个资源类型返回是 null 的情况，这种情况下JAX-RS 框架一般会返回状态204，表示请求处理成功但没有响应内容。 我们对这种情况也重新处理改为响应为 404。

- 然后，对JAX-RS使用方客户端使用一个filter,暗示不要对结果进行封装处理：

```
@Provider
public class UnwrapResultJaxrsFilter implements ClientRequestFilter {
    @Override
    public void filter(ClientRequestContext clientRequestContext) throws IOException {
        clientRequestContext.getHeaders().add("Not-Wrap-Result", "T");
    }
}
```
通过上述方法，我们可以开发代码更简单点，远离恼人的结构封装了，将封装逻辑作为一个全局的可协商行为，封装逻辑方便变更。虽然封装一般不建议轻易变更，会考虑对其他已有用户的支持和兼容，但可以让不同用户客户端通过 header 暗示封装的（类型)版本做到差异化对待。

- 服务器端，也许个别请求是明确需要（或者不需要）结果封装怎么办？

最简单的方法，可以在服务方法体内容，将 Request 的"wrap-result"情况，或者是指定为 True;

```
@Context HttpRequest resteasyHttpRequest;

public Movie movieByImdbId( String imdbId){
  resteasyHttpRequest.remove("wrap-result");
}

当然也可以自定义个注解"@ResultWrapper(false)"，不要写上述代码，如下：

```
@GET
@Path("/getinfo")
@Produces({ MediaType.APPLICATION_JSON, MediaType.APPLICATION_XML })
@ResultWrapper(false)
Movie movieByImdbId(@QueryParam("imdbId") String imdbId);
```


---
### 著作权声明

`首次发布于此，转载请保留以上链接`
