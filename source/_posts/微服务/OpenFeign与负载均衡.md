---
title: OpenFeign与负载均衡
date: 2023-03-01 10:22:42
categories:
    - 微服务
tags:
    - OpenFeign
    - 负载均衡
---

## 一、Ribbon与OpenFeign关系

　　说到 OpenFeign就不得不提 Ribbon，OpenFeign默认将Ribbon作为负载均衡器，直接内置了 Ribbon。在导入OpenFeign 依赖后无需专门导入Ribbon 依赖。

　　Ribbon 是 Netflix 公司的一个开源的负载均衡项目，一个客户端负载均衡器，运行在消费者端。简单来说就是在消费者端配置对提供者的负载均衡器。这点与 Dubbo略有不同，Dubbo 在消费者端与提供者端均可配置负载均衡器。

## 二、声明式Rest客户端OpenFeign案例



### （一）消费端配置

　　1、添加openfeign依赖

　　　　注意， 这里使用的是 spring-cloud-starter-openfeign 依赖，而非 spring-cloud-starter-feign依赖。

```
 		<!--feign依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
            <version>2.2.3.RELEASE</version>
        </dependency>
```

　　2、定义feign接口

　　　　使用@FeignClient设置需要调用的项目名称，同时使用RequestMapping进行调用，调用地址要和服务提供者的地址保持一致。

```
@FeignClient(value = "provider01-depart")//注解作用：声明当前为Feign客户端接口
@RequestMapping("/provider/depart")// 参数为要调用的提供者相应的uri，抽取所有方法的公有uri地址
public interface DepartService {//更加符合面向接口api调用习惯
    @PostMapping("/save")
    boolean saveDepart(@RequestBody Depart depart);
    @DeleteMapping("/del/{id}")
    boolean removeDepartById(@PathVariable("id") int id);
    @PutMapping("/update")
    boolean modifyDepart(@RequestBody Depart depart);
    @GetMapping("/get/{id}")
    Depart getDepartById(@PathVariable("id") int id);
    @GetMapping("/list")
    List<Depart> listAllDeparts();
}
```

　　3、处理器注入Feign客户端对象

```
@RestController
@RequestMapping("/feign/consumer/depart")
public class DepartFeignController {
    @Autowired
    private DepartService departService;//跨服务根据id查询
    @GetMapping("/get/{id}")
    public Depart getHandle(@PathVariable("id") int id) {
        return departService.getDepartById(id);
    }
}
```

　　4、修改启动类

　　　　使用@EnableFeignClients来打开对Feign客户端的支持，同时设置客户端扫描的包。

　　　　由于直接使用了Feign Client，因此就不再使用RestTemplate进行处理了。

```
@SpringBootApplication
@EnableFeignClients(basePackages = "com.lcl.cloud.alibaba.consumer02.service")//开启当前服务支持Feign客户端，作用扫描所有客户端接口
public class Consumer02Application {

    public static void main(String[] args) {
        SpringApplication.run(Consumer02Application.class, args);
    }

//    @Bean
//    @LoadBalanced
//    public RestTemplate restTemplate() {
//        return new RestTemplate();
//    }

}
```

　　5、需要注意的一点，就是Feign的服务名称不能使用下划线和横线，否则会报 Service id not legal hostname (provider02_nacosconfig) 的错误。

![img](OpenFeign与负载均衡/1782729-20211014215359284-507267638.png)

### （二）超时配置

　　1、客户端的超时配置

```
feign:
  client:
    config:
      default:
        #连接超时时间
        connectTimeout: 5000
        #数据读取超时是时间
        readTimeout: 5000
```

 　2、服务端模拟超时　　

```
 	public Depart getDepartById(int id) {
        try {
            TimeUnit.SECONDS.sleep(6);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if(repository.existsById(id)) {
            return repository.getOne(id);
        }
        Depart depart = new Depart();
        depart.setName(departName);
        return depart;
    }
```

　　3、验证结果

![img](OpenFeign与负载均衡/1782729-20211014220110841-1283013534.png)

###  （三）Gzip压缩设置

　　Feign 支持对请求和响应进行 Gzip 压缩以提高通信效率。注意，这里的请求是指 Feign向提供者所提交的请求，响应是指 Feign 向客户端作出的响应。

![img](OpenFeign与负载均衡/1782729-20211014220305776-594987007.png)

　　配置在官网中可以查看：https://docs.spring.io/spring-cloud-openfeign/docs/2.2.5.RELEASE/reference/html/#feign-requestresponse-compression

　　参数如下所示：

```
feign.compression.request.enabled=true
feign.compression.response.enabled=true
```

## 三、更换负载均衡策略

### （一）更换内置策略

　　若要更换负载均衡策略，则首先要了解负载均衡策略的定义接口 IRule。Ribbon 默认采用的是 RoundRobinRule，即轮询策略。但通过修改消费者工程的配置文件，或修改消费者的启动类或 JavaConfig 类可以实现更换负载均衡策略的目的。

　　1、修改配置文件

　　　　修改配置文件，在其中添加如下内容，指定要使用的负载均衡策略 <clientName>. <clientConfigNameSpace>.NFLoadBalancerRuleClassName。

　　　　该方式的好处是，可以为不同的微服务指定相应的负载均衡策略。

```
provider02Nacosconfig:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

 

　　2、修改JavaConfig类

　　　　在 JavaConfig 类中添加负载 Bean 方法。全局所有feign对应服务都可以生效。 

```
@Configuration
public class FeignConfiguration {
    /**
     * 配置随机的负载均衡策略
     * 特点：对所有的服务都生效
     */
    @Bean
    public IRule loadBalancedRule() {
        return new RandomRule();
    }
}
```

### （二）自定义负载均衡策略

　　1、定义一个负载均衡策略

　　　　该负载均衡策略的思路是：从所有可用的 provider 中排除掉指定端口号的provider，剩余 provider 进行随机选择。 

```
/**
 * 自定义负载均衡算法
 */
public class CustomRule implements IRule {
    private ILoadBalancer lb;
    private List<Integer> excludePorts;

    public CustomRule() {
    }

    public CustomRule(List<Integer> excludePorts) {
        this.excludePorts = excludePorts;
    }

    @Override
    public void setLoadBalancer(ILoadBalancer lb) {
        this.lb = lb;
    }

    @Override
    public ILoadBalancer getLoadBalancer() {
        return lb;
    }

    /**
     * 目标：自定义负载均衡策略：从所有可用的provider中排除掉指定端口号的provider，剩余provider进行随机选择
     * 实现步骤：
     * 1.获取到所有Server
     * 2.从所有Server中排除掉指定端口的Server后，剩余的Server
     * 3.从剩余Server中随机选择一个Server
     */
    @Override
    public Server choose(Object key) {
        // 1.获取到所有Server
        List<Server> servers = lb.getReachableServers();
        // 2.从所有Server中排除掉指定端口的Server后，剩余的Server
        List<Server> availableServers = this.getAvailableServers(servers);
        // 3.从剩余Server中随机选择一个Server
        return this.getAvailableRandomServers(availableServers);
    }

    private List<Server> getAvailableServers(List<Server> servers) {
        // 若没有指定要排除的port，则返回所有Server
        if(excludePorts == null || excludePorts.size() == 0) {
            return servers;
        }
        List<Server> aservers = servers.stream()
                // filter()
                // noneMatch() 只有当流中所有元素都没有匹配上时，才返回true，只要有一个匹配上了，则返回false
                .filter(server -> excludePorts.stream().noneMatch(port -> server.getPort() == port))
                .collect(Collectors.toList());

        return aservers;
    }

    private Server getAvailableRandomServers(List<Server> availableServers) {
        // 获取一个[0,availableServers.size())的随机数
        int index = new Random().nextInt(availableServers.size());
        return availableServers.get(index);
    }
}
```

　　2、修改JavaConfig类，使用自定义的负载均衡策略

```
@Configuration
public class FeignConfiguration {
    @Bean
    public IRule loadBalancedRule() {
        List<Integer> list = new ArrayList<>();
        list.add(8081);//排除访问端口
        return new CustomRule(list);
    }
}
```

### （三）Ribbon内置负载均衡算法

　　1、 RoundRobinRule

　　　　轮询策略：Ribbon 默认采用的策略。若经过一轮轮询没有找到可用的provider，其最多轮询 10 轮（代码中写死的，不能修改）。若还未找到，则返回 null。

　　2、RandomRule

　　　　随机策略：从所有可用的 provider 中随机选择一个。

　　3、RetryRule

　　　　重试策略：先按照 RoundRobinRule 策略获取 server，若获取失败，则在指定的时限内重试。默认的时限为 500 毫秒。

　　4、BestAvailableRule

　　　　最可用策略：选择并发量最小的 provider，即连接的消费者数量最少的provider。其会遍历服务列表中的每一个server，选择当前连接数量minimalConcurrentConnections 最小的server。

　　5、AvailabilityFilteringRule

　　　　可用过滤算法：该算法规则是过滤掉处于熔断状态的 server 与已经超过连接极限的server，对剩余 server 采用轮询策略。 

## 四、负载均衡器SpringCloudLoadBalancer 

　　由于 Netflix 对于 Ribbon 的维护已经暂停，所以 Spring Cloud 对于负载均衡建议使用由其自己定义的 Spring Cloud LoadBalancer。对于Spring Cloud LoadBalancer 的使用非常简单。

　　1、关闭Ribbon的负载均衡器

```
spring:
  application:
    name: consumer01-depart
  cloud:
    loadbalancer:
      # 关闭Ribbon的负载均衡器
      ribbon:
        enabled: false
```

　　2、pom中添加LoadBalancer依赖

```
		<!--spring cloud loadbalancer 依赖--> 
        <dependency> 
            <groupId>org.springframework.cloud</groupId> 
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
            <version>2.2.3.RELEASE</version>
        </dependency>
```

