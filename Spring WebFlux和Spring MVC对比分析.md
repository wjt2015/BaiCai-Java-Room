> 白菜Java自习室 涵盖核心知识

## 一、什么是 Spring WebFlux

下图截自 Spring Boot 官方网站：
![](https://user-gold-cdn.xitu.io/2020/6/17/172c16b9eaf50dc4?w=1478&h=1240&f=jpeg&s=288191)

结合上图，在了解 Spring WebFlux 之前，我们先来对比说说什么是 Spring MVC，这更有益我们去理解 WebFlux，图右边对 Spring MVC 的定义，原文如下：

> Spring MVC is built on the Servlet API and uses a synchronous blocking I/O architecture whth a one-request-per-thread model.</br>

**Spring MVC 构建于 Servlet API 之上，使用的是同步阻塞式 I/O 模型，什么是同步阻塞式 I/O 模型呢？就是说，每一个请求对应一个线程去处理**。

了解了 Spring MVC 之后，再来说说 Spring WebFlux:

上图左边，官方给出的定义如下：

> Spring WebFlux is a non-blocking web framework built from the ground up to take advantage of multi-core, next-generation processors and handle massive numbers of concurrent connections.

**Spring WebFlux 是一个异步非阻塞式的 Web 框架，它能够充分利用多核 CPU 的硬件资源去处理大量的并发请求**。

## 二、WebFlux 的优势&提升性能?

WebFlux 内部使用的是响应式编程（Reactive Programming），以 Reactor 库为基础, 基于异步和事件驱动，可以让我们在不扩充硬件资源的前提下，提升系统的吞吐量和伸缩性。

看到这里，你是不是以为 WebFlux 能够使程序运行的更快呢？量化一点，比如说我使用 WebFlux 以后，一个接口的请求响应时间是不是就缩短了呢？

**抱歉了，答案是否定的**！以下是官方原话：

> Reactive and non-blocking generally do not make applications run faster.

**WebFlux 并不能使接口的响应时间缩短，它仅仅能够提升吞吐量和伸缩性**。

## 三、WebFlux 应用场景

上面说到了， Spring WebFlux 是一个异步非阻塞式的 Web 框架，所以，它特别适合应用在 IO 密集型的服务中，比如微服务网关这样的应用中。

> PS: IO 密集型包括：**磁盘IO密集型, 网络IO密集型**，微服务网关就属于网络 IO 密集型，使用异步非阻塞式编程模型，能够显著地提升网关对下游服务转发的吞吐量。
![](https://user-gold-cdn.xitu.io/2020/6/17/172c172b2a8ee498?w=1574&h=1056&f=jpeg&s=188822)

## 四、选 WebFlux 还是 Spring MVC?

首先你需要明确一点就是：**WebFlux 不是 Spring MVC 的替代方案**！，虽然 WebFlux 也可以被运行在 Servlet 容器上（需是 Servlet 3.1+ 以上的容器），但是 WebFlux 主要还是应用在异步非阻塞编程模型，而 Spring MVC 是同步阻塞的，如果你目前在 Spring MVC 框架中大量使用非同步方案，那么，WebFlux 才是你想要的，否则，使用 Spring MVC 才是你的首选。

在微服务架构中，Spring MVC 和 WebFlux 可以混合使用，比如已经提到的，对于那些 IO 密集型服务(如网关)，我们就可以使用 WebFlux 来实现。

**选 WebFlux 还是 Spring MVC? This is not a problem！**

咱不能为了装逼而装逼，为了技术而技术，还要考量到转向非阻塞响应式编程学习曲线是陡峭的，小组成员的学习成本等诸多因素。

**总之一句话，在合适的场景中，选型最合适的技术。**

## 五、异同点

![](https://user-gold-cdn.xitu.io/2020/6/17/172c174b209191bc?w=800&h=446&f=png&s=100834)

从上图中，可以一眼看出 Spring MVC 和 Spring WebFlux 的相同点和不同点：

**相同点**：

* 都可以使用 Spring MVC 注解，如 `@Controller`, 方便我们在两个 Web 框架中自由转换；
* 均可以使用 Tomcat, Jetty, Undertow Servlet 容器（Servlet 3.1+）；

**注意点**：

* Spring MVC 因为是使用的同步阻塞式，更方便开发人员编写功能代码，Debug 测试等，一般来说，如果 Spring MVC 能够满足的场景，就尽量不要用 WebFlux;
* WebFlux 默认情况下使用 Netty 作为服务器;
* WebFlux 不支持 MySql;

## 六、简单看看 WebFlux 是如何分发请求的

使用过 Spring MVC 的小伙伴们，应该到知道 Spring MVC 的前端控制器是 DispatcherServlet, 而 WebFlux 是 DispatcherHandler，它实现了 WebHandler 接口：
```
public interface WebHandler {
    Mono<Void> handle(ServerWebExchange var1);
}
```
来看看DispatcherHandler类中处理请求的 handle 方法：
```
    public Mono<Void> handle(ServerWebExchange exchange) {
        return this.handlerMappings == null ? this.createNotFoundError() : Flux.fromIterable(this.handlerMappings).concatMap((mapping) -> {
            return mapping.getHandler(exchange);
        }).next().switchIfEmpty(this.createNotFoundError()).flatMap((handler) -> {
            return this.invokeHandler(exchange, handler);
        }).flatMap((result) -> {
            return this.handleResult(exchange, result);
        });
    }
```
* ①：ServerWebExchange 对象中放置每一次 HTTP 请求响应信息，包括参数等；
* ②：判断整个接口映射 mappings 集合是否为空，空则创建一个 Not Found 的错误；
* ③：根据具体的请求地址获取对应的 handlerMapping;
* ④：调用具体业务方法，也就是我们定义的接口方法；
* ⑤：处理返回的结果；

## 七、快速入门

#### 7.1 添加 webflux 依赖

新建一个 Spring Boot 项目，在 pom.xml 文件中添加 webflux 依赖：
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

#### 7.2 定义接口

新建一个 controller 包，用来放置对外的接口类，再创建一个 WebFluxController 类，定义两个接口：
```
@RestController
public class WebFluxController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello, WebFlux !";
    }

    @GetMapping("/user")
    public Mono<UserDO> getUser() {
        UserDO user = new UserDO();
        user.setName("白菜");
        user.setDesc("白菜Java自习室");
        return Mono.just(user);
    }

}

```
```
@Data
public class UserDO {

    /**
     * 姓名
     */
    private String name;
    /**
     * 描述
     */
    private String desc;

}
```
以上控制器类中，我们使用的全都是 Spring MVC 的注解，分别定义了两个接口：

* 一个 GET 请求的 `/hello` 接口，返回 Hello, WebFlux !字符串。
* 又定义了一个 GET 请求的 `/user`方法，返回的是 JSON 格式 User 对象。

这里注意，User 对象是通过 Mono 对象包装的，你可能会问，为啥不直接返回呢？

在 WebFlux 中，Mono 是非阻塞的写法，只有这样，你才能发挥 WebFlux 非阻塞 + 异步的特性。

> 在 WebFlux 中，除了 Mono 外，还有一个 Flux，不同的是：
> * Mono：返回 0 或 1 个元素，即单个对象。
> * Flux：返回 N 个元素，即 List 列表对象。

#### 7.3 测试接口

启动项目，查看控制台输出：
![](https://user-gold-cdn.xitu.io/2020/6/17/172c28b65608c4ba?w=1662&h=355&f=png&s=29724)

当控制台中输出中包含 Netty started on port(s): 8080 语句时，说明默认使用 Netty 服务已经启动了。

先对 `/hello` 接口发起调用：
```
GET http://localhost:8080/hello
Accept: application/json
```
```
GET http://localhost:8080/hello

HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Content-Length: 16

Hello, WebFlux !

Response code: 200 (OK); Time: 681ms; Content length: 16 bytes
```
再来对 `/user` 接口测试一下:
```
GET http://localhost:8080/user
Accept: application/json
```
```
GET http://localhost:8080/user

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 46

{
  "name": "白菜",
  "desc": "白菜Java自习室"
}

Response code: 200 (OK); Time: 148ms; Content length: 32 bytes
```