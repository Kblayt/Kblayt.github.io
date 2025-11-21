---
title: 微服务调用组件Feign
date: 2024-03-07 10:27:32
categories:
    - 系统架构
    - 架构之微服务组件
    - Feign
tags:
    - Feign
---

**1. RPC概述**

思考： 微服务之间如何方便优雅的实现服务间的远程调用？

RPC 全称是 Remote Procedure Call ，即远程过程调用，其对应的是我们的本地调用。RPC 的目的是：让我们调用远程方法像调用本地方法一样。

```
//本地调用
R result = orderService.findOrderByUserId(id);
//RPC远程调用  orderService为代理对象
R result = orderService.findOrderByUserId(id);
```

​      

**RPC框架设计架构**

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53802)

**2. 什么是Feign**

Feign是Netflix开发的声明式、模板化的HTTP客户端，Feign可帮助我们更加便捷、优雅地调用HTTP API。

Feign可以做到使用 HTTP 请求远程服务时就像调用本地方法一样的体验，开发者完全感知不到这是远程方法，更感知不到这是个 HTTP 请求。它像 Dubbo 一样，consumer 直接调用接口方法调用 provider，而不需要通过常规的 Http Client 构造请求再解析返回数据。它解决了让开发者调用远程接口就跟调用本地方法一样，无需关注与远程的交互细节，更无需关注分布式环境开发。

Spring Cloud openfeign对Feign进行了增强，使其支持Spring MVC注解，另外还整合了Ribbon和Eureka，从而使得Feign的使用更加方便。

**2.1 Ribbon&Feign对比**

 **Ribbon+RestTemplate进行微服务调用**

```
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

//调用方式
String url = "http://mall-order/order/findOrderByUserId/"+id;
R result = restTemplate.getForObject(url,R.class);
```

​    

**Feign进行微服务调用**

```
@FeignClient(value = "mall-order",path = "/order")
public interface OrderFeignService {
    @RequestMapping("/findOrderByUserId/{userId}")
    public R findOrderByUserId(@PathVariable("userId") Integer userId);
}

@Autowired
OrderFeignService orderFeignService;
//feign调用
R result = orderFeignService.findOrderByUserId(id);
```

​        

**2.2 Feign的设计架构**

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/54803)

**2.3 Spring Cloud Alibaba快速整合Feign**

**1）引入依赖**

```
<!-- openfeign 远程调用 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

**2）编写调用接口+@FeignClient注解**

```
@FeignClient(value = "mall-order",path = "/order")
public interface OrderFeignService {

    @RequestMapping("/findOrderByUserId/{userId}")
    public R findOrderByUserId(@PathVariable("userId") Integer userId);
}
```

​          

**3）调用端在启动类上添加****@EnableFeignClients****注解**

```
@SpringBootApplication
@EnableFeignClients  //扫描和注册feign客户端的beanDefinition
public class MallUserFeignDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(MallUserFeignDemoApplication.class, args);
    }
}
```

​     

**4）发起调用，像调用本地方式一样调用远程服务**

```
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    OrderFeignService orderFeignService;

    @RequestMapping(value = "/findOrderByUserId/{id}")
    public R  findOrderByUserId(@PathVariable("id") Integer id) {
        //feign调用
        R result = orderFeignService.findOrderByUserId(id);
        return result;
    }
}
```

提示： Feign 的继承特性可以让服务的接口定义单独抽出来，作为公共的依赖，以方便使用。

**2.3 Spring Cloud Feign扩展**

 Feign 提供了很多的扩展机制，让用户可以更加灵活的使用。

**日志配置**

有时候我们遇到 Bug，比如接口调用失败、参数没收到等问题，或者想看看调用性能，就需要配置 Feign 的日志了，以此让 Feign 把请求信息输出来。

**1）定义一个配置类，指定日志级别**

```
// 注意： 此处配置@Configuration注解就会全局生效，如果想指定对应微服务生效，就不能配置@Configuration
@Configuration
public class FeignConfig {
    /**
     * 日志级别
     *
     * @return
     */
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

​       

通过源码可以看到日志等级有 4 种，分别是：

- **NONE**【性能最佳，适用于生产】：不记录任何日志（默认值）。
- **BASIC**【适用于生产环境追踪问题】：仅记录请求方法、URL、响应状态代码以及执行时间。
- **HEADERS**：记录BASIC级别的基础上，记录请求和响应的header。
- **FULL**【比较适用于开发及测试环境定位问题】：记录请求和响应的header、body和元数据。

**2) 局部配置，让调用的微服务生效，在****@FeignClient 注解中指定使用的配置类**

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53054)

**3) 在yml配置文件中配置 Client 的日志级别才能正常输出日志，格式是"logging.level.feign接口包路径=debug"**

```
logging:
  level:
    com.tuling.mall.feigndemo.feign: debug
```

​     

测试：BASIC级别日志

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53060)

**补充：局部配置可以在yml中配置**

对应属性配置类： org.springframework.cloud.openfeign.FeignClientProperties.FeignClientConfiguration

```
feign:
  client:
    config:
      mall-order:  #对应微服务
        loggerLevel: FULL
```

​        

**契约配置**

Spring Cloud 在 Feign 的基础上做了扩展，可以让 Feign 支持 Spring MVC 的注解来调用。原生的 Feign 是不支持 Spring MVC 注解的，如果你想在 Spring Cloud 中使用原生的注解方式来定义客户端也是可以的，通过配置契约来改变这个配置，Spring Cloud 中默认的是 SpringMvcContract。

**1）修改契约配置，支持Feign原生的注解**

```
/**
 * 修改契约配置，支持Feign原生的注解
 * @return
 */
@Bean
public Contract feignContract() {
    return new Contract.Default();
}
```

​      

注意：修改契约配置后，OrderFeignService 不再支持springmvc的注解，需要使用Feign原生的注解

**2）OrderFeignService 中配置使用Feign原生的注解**

```
@FeignClient(value = "mall-order",path = "/order")
public interface OrderFeignService {
    @RequestLine("GET /findOrderByUserId/{userId}")
    public R findOrderByUserId(@Param("userId") Integer userId);
}
```

​       

**3）补充，也可以通过yml配置契约**

```
feign:
  client:
    config:
      mall-order:  #对应微服务
        loggerLevel: FULL
        contract: feign.Contract.Default   #指定Feign原生注解契约配置
```

​    

**通过拦截器实现参数传递**

通常我们调用的接口都是有权限控制的，很多时候可能认证的值是通过参数去传递的，还有就是通过请求头去传递认证信息，比如 Basic 认证方式。  

**Feign 中我们可以直接配置 Basic 认证**

```
@Configuration  // 全局配置
public class FeignConfig {
    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor("fox", "123456");
    }
}
```

​       

**扩展点： feign.RequestInterceptor** 

每次 feign 发起http调用之前，会去执行拦截器中的逻辑。

```
public interface RequestInterceptor {

  /**
   * Called for every request. Add data using methods on the supplied {@link RequestTemplate}.
   */
  void apply(RequestTemplate template);
}
```

使用场景

1. 统一添加 header 信息；
2. 对 body 中的信息做修改或替换；

**自定义拦截器实现认证逻辑**

```
public class FeignAuthRequestInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        // 业务逻辑
        String access_token = UUID.randomUUID().toString();
        template.header("Authorization",access_token);
    }
}

@Configuration  // 全局配置
public class FeignConfig {
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
    /**
     * 自定义拦截器
     * @return
     */
    @Bean
    public FeignAuthRequestInterceptor feignAuthRequestInterceptor(){
        return new FeignAuthRequestInterceptor();
    }
}
```

​    

测试

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53055)

**补充：可以在yml中配置**

```
feign:
  client:
    config:
      mall-order:  #对应微服务
        requestInterceptors[0]:  #配置拦截器
          com.tuling.mall.feigndemo.interceptor.FeignAuthRequestInterceptor
```

mall-order端可以通过 @RequestHeader获取请求参数，建议在filter,interceptor中处理

**超时时间配置**

通过 Options 可以配置连接超时时间和读取超时时间，Options 的第一个参数是连接的超时时间（ms），默认值是 2s；第二个是请求处理的超时时间（ms），默认值是 5s。

**全局配置**

```
@Configuration
public class FeignConfig {
    @Bean
    public Request.Options options() {
        return new Request.Options(5000, 10000);
    }
}
```

​      

**yml中配置**

```
feign:
  client:
    config:
      mall-order:  #对应微服务
        # 连接超时时间，默认2s
        connectTimeout: 5000
        # 请求处理超时时间，默认5s
        readTimeout: 10000
```

补充说明： Feign的底层用的是Ribbon，但超时时间以Feign配置为准

测试超时情况：

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53049)

返回结果

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53056)

**客户端组件配置**

Feign 中默认使用 JDK 原生的 URLConnection 发送 HTTP 请求，我们可以集成别的组件来替换掉 URLConnection，比如 Apache HttpClient，OkHttp。

Feign发起调用真正执行逻辑：**feign.Client#execute   （扩展点）**

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53053)

**配置Apache HttpClient**

引入依赖

```
<!-- Apache HttpClient -->
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.7</version>
</dependency>
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>10.1.0</version>
</dependency>
```

​         

**然后修改yml配置，将 Feign 的** **Apache HttpClient****启用 ：**

```
feign:
  #feign 使用 Apache HttpClient  可以忽略，默认开启
  httpclient:
    enabled: true  
```

​      

关于配置可参考源码： org.springframework.cloud.openfeign.FeignAutoConfiguration

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53044)

测试：调用会进入feign.httpclient.ApacheHttpClient#execute

**配置 OkHttp**

引入依赖

```
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

然后修改yml配置，将 Feign 的 HttpClient 禁用，启用 OkHttp，配置如下：

```
feign:
  #feign 使用 okhttp
  httpclient:
    enabled: false
  okhttp:
    enabled: true
```

   

关于配置可参考源码： org.springframework.cloud.openfeign.FeignAutoConfiguration

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53048)

测试：调用会进入feign.okhttp.OkHttpClient#execute

**GZIP 压缩配置**

开启压缩可以有效节约网络资源，提升接口性能，我们可以配置 GZIP 来压缩数据：

```
feign:
  # 配置 GZIP 来压缩数据
  compression:
    request:
      enabled: true
      # 配置压缩的类型
      mime-types: text/xml,application/xml,application/json
      # 最小压缩值
      min-request-size: 2048
    response:
      enabled: true
```

​     

注意：只有当 Feign 的 Http Client 不是 okhttp3 的时候，压缩才会生效，配置源码在FeignAcceptGzipEncodingAutoConfiguration

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53045)

核心代码就是 @ConditionalOnMissingBean（type="okhttp3.OkHttpClient"），表示 Spring BeanFactory 中不包含指定的 bean 时条件匹配，也就是没有启用 okhttp3 时才会进行压缩配置。

**编码器解码器配置**

Feign 中提供了自定义的编码解码器设置，同时也提供了多种编码器的实现，比如 Gson、Jaxb、Jackson。我们可以用不同的编码解码器来处理数据的传输。如果你想传输 XML 格式的数据，可以自定义 XML 编码解码器来实现获取使用官方提供的 Jaxb。

扩展点：Encoder & Decoder 

```
public interface Encoder {
    void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException;
}
public interface Decoder {
    Object decode(Response response, Type type) throws IOException, DecodeException, FeignException;
}
```

  

**Java配置方式**

配置编码解码器只需要在 Feign 的配置类中注册 Decoder 和 Encoder 这两个类即可:

```
@Bean
public Decoder decoder() {
    return new JacksonDecoder();
}
@Bean
public Encoder encoder() {
    return new JacksonEncoder();
}
```

**yml配置方式**

```
feign:
  client:
    config:
      mall-order:  #对应微服务
        # 配置编解码器
        encoder: feign.jackson.JacksonEncoder
        decoder: feign.jackson.JacksonDecoder
```

  

**3. Spring Cloud整合Dubbo**

**3.1 provider端配置**

 **1）引入依赖**

```
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-dubbo</artifactId>
    <version>2.2.7.RELEASE</version>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

注意：因为spring cloud alibaba 2.2.8这个版本没有整合dubbo，所以需要指定dubbo的版本

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/54856)

**2）修改application.yml**

```
dubbo:
  scan:
    # 指定 Dubbo 服务实现类的扫描基准包
    base-packages: com.tuling.mall.user.service
#  application:
#    name: ${spring.application.name}
  protocol:
    # dubbo 协议
    name: dubbo
    # dubbo 协议端口（ -1 表示自增端口，从 20880 开始）
    port: -1
#  registry:
#    #挂载到 Spring Cloud 注册中心  高版本可选
#    address: spring-cloud://127.0.0.1:8848

spring:
  application:
    name: spring-cloud-dubbo-provider-user
  main:
    # Spring Boot2.1及更高的版本需要设定
    allow-bean-definition-overriding: true
  cloud:
    nacos:
      # Nacos 服务发现与注册配置
      discovery:
        server-addr: 127.0.0.1:8848
```

​      

**3）服务实现类上配置****@DubboService****暴露服务**

```
@DubboService
public class UserServiceImpl implements UserService {

   @Autowired
   private UserMapper userMapper;

   @Override
   public List<User> list() {
      return userMapper.list();
   }

   @Override
   public User getById(Integer id) {
      return userMapper.getById(id);
   }
}
```

​          

**3.2 consumer端配置**

**1）引入依赖**

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-dubbo</artifactId>
    <version>2.2.7.RELEASE</version>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

**2）修改application.yml**

```
dubbo:
  cloud:
    # 指定需要订阅的服务提供方，默认值*，会订阅所有服务，不建议使用
    subscribed-services: spring-cloud-dubbo-provider-user
#  application:
#    name: ${spring.application.name}
  protocol:
    # dubbo 协议
    name: dubbo
    # dubbo 协议端口（ -1 表示自增端口，从 20880 开始）
    port: -1
#  registry:
#    #挂载到 Spring Cloud 注册中心  高版本可选
#    address: spring-cloud://127.0.0.1:8848

spring:
  application:
    name: spring-cloud-dubbo-consumer-user
  main:
    # Spring Boot2.1及更高的版本需要设定
    allow-bean-definition-overriding: true
  cloud:
    nacos:
      # Nacos 服务发现与注册配置
      discovery:
        server-addr: 127.0.0.1:8848
```

当应用使用属性dubbo.cloud.subscribed-services为默认值时，日志中将会输出警告：

​    ![0](%E5%BE%AE%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8%E7%BB%84%E4%BB%B6Feign/53843)

**3）服务消费方通过****@DubboReference****引入服务**

```
@RestController
@RequestMapping("/user")
public class UserConstroller {

    @DubboReference
    private UserService userService;

    @RequestMapping("/info/{id}")
    public User info(@PathVariable("id") Integer id){

        return userService.getById(id);
    }

    @RequestMapping("/list")
    public List<User> list(){

        return userService.list();
    }
}
```

​       

**3.3 从Open Feign迁移到Dubbo**

Dubbo Spring Cloud 提供了方案，可以从Open Feign迁移到Dubbo，即 @DubboTransported 注解。能够帮助服务消费端的 Spring Cloud Open Feign 接口以及 @LoadBalanced RestTemplate Bean 底层走 Dubbo 调用（可切换 Dubbo 支持的协议），而服务提供方则只需在原有 @RestController 类上追加 Dubbo @Servce 注解（需要抽取接口）即可，换言之，在不调整 Feign 接口以及 RestTemplate URL 的前提下，实现无缝迁移。

**1）修改服务提供者**

```
@DubboService
@Slf4j
@RestController
@RequestMapping("/user")
public class UserServiceImpl implements UserService {

   @Autowired
   private UserMapper userMapper;

   @Override
   @RequestMapping("/list")
   public List<User> list() {
      log.info("查询user列表");
      return userMapper.list();
   }

   @Override
   @RequestMapping("/getById/{id}")
   public User getById(@PathVariable("id") Integer id) {
      return userMapper.getById(id);
   }
}
```

​     

**2）服务消费端引入依赖**

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-dubbo</artifactId>
    <version>2.2.7.RELEASE</version>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

​        

**3）添加Feign的实现，启动类上添加****@EnableFeignClients**

```
@SpringBootApplication
@EnableFeignClients
public class SpringCloudDubboConsumerUserFeignApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudDubboConsumerUserFeignApplication.class, args);
    }

}
```

**4）Feign接口添加** **@DubboTransported** **注解**

```
@FeignClient(value = "spring-cloud-dubbo-provider-user-feign",path = "/user")
@DubboTransported(protocol = "dubbo")
public interface UserDubboFeignService {

    @RequestMapping("/list")
    public List<User> list();

    @RequestMapping("/getById/{id}")
    public User getById(@PathVariable("id") Integer id);
}

@FeignClient(value = "spring-cloud-dubbo-provider-user-feign",path = "/user")
public interface UserFeignService {

    @RequestMapping("/list")
    public List<User> list();


    @RequestMapping("/getById/{id}")
    public User getById(@PathVariable("id") Integer id);
}
```

​         

**5）调用对象添加** **@DubboTransported** **注解，发起调用**

```
@RestController
@RequestMapping("/user")
public class UserConstroller {

    @DubboReference
    private UserService userService;

    @RequestMapping("/info/{id}")
    public User info(@PathVariable("id") Integer id){
        return userService.getById(id);
    }

    @Autowired
    private UserFeignService userFeignService;

    @RequestMapping("/list")
    public List<User> list(){
        return userFeignService.list();
    }

    @Autowired
    private UserDubboFeignService userDubboFeignService;

    @RequestMapping("/list2")
    public List<User> list2(){

        return userDubboFeignService.list();
    }

}
```

