---
layout: post
categories: 整理记录
author: chen
title: SpringCloudAlibaba分布式组件整理
---
![image-20251206180112908](https://raw.githubusercontent.com/chen-1110/image/main/image-20251206180112908.png)

![image-20251206180130802](https://raw.githubusercontent.com/chen-1110/image/main/image-20251206180130802.png)

## nacos

服务注册：java服务启动时上报nacos自身ip端口，nacos记录

服务发现：A调用B服务，A服务调用nacos获取B服务的ip端口列表，负载均衡选择其一调用

配置中心：记录服务配置，可通过@RefeshScope自动刷新； 配置中心通过命名空间（区分生产测试），服务分组（区分各自微服务），数据集dataId（区分同一服务中多个配置文件）来做配置数据隔离

![image-20251206180609703](https://raw.githubusercontent.com/chen-1110/image/main/image-20251206180609703.png)

## OpenFeign

注解驱动声明式REST客户端

• 指定远程地址：@FeignClient

• 指定请求方式：@GetMapping、@PostMapping、@DeleteMapping ...

• 指定携带数据：@RequestHeader、@RequestParam、@RequestBody ...

• 指定结果返回：响应模型

```
@FeignClient(value = "service-product", fallback = ProductFeignClientFallback.class)
public interface ProductServiceFeignClient {

    @GetMapping("sleep")
    public String testSleep();

    @GetMapping("product/{id}")
    public String getProductById(@PathVariable("id") Long id);
}

```

### 用法

对于注册中心内部服务，通过value指定服务名，从注册中心取到服务ip列表后，自动负载均衡调用

![image-20251206180957537](https://raw.githubusercontent.com/chen-1110/image/main/image-20251206180957537.png)

对于外部api/k8s内其他service，通过url指定远程调用地址，无法实现负载均衡，服务端自行实现

### 进阶功能

超时控制：默认连接超时10s，逻辑执行超时60s。可修改配置自行设置超时时间

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            logger-level: full
            connect-timeout: 1000
            read-timeout: 2000
```

重试：可创建Retry bean类实现阶梯重试机制

```
    @Bean
    Retryer retryer() {
        return new Retryer.Default();
    }
```

兜底机制：对于错误/超时请求，可设置兜底逻辑返回默认数据。（需整合sentinel）

```
@FeignClient(value = "service-product", fallback = ProductFeignClientFallback.class)
public interface ProductServiceFeignClient {
    @GetMapping("product/{id}")
    public String getProductById(@PathVariable("id") Long id);
}

@Component
public class ProductFeignClientFallback implements ProductServiceFeignClient {
    @Override
    public String getProductById(Long id) {
        return "getProductById fallback";
    }
}
```

![image-20251206182000632](https://raw.githubusercontent.com/chen-1110/image/main/image-20251206182000632.png)

## sentinel

资源与规则：每个远程调用/方法/接口都是资源，可以针对资源定义不同的规则，实现流控，熔断，热点问题处理等高可用实现。

sentinel是微服务调用中客户端实现用来保护客户端（调用）服务稳定性的方案

![image-20251207033419574](https://raw.githubusercontent.com/chen-1110/image/main/image-20251207033419574.png)

### 使用方式

1.确认目标资源是什么，用注解等方式标注资源

2.在sentinel控制台对目标资源创建规则

3.（可选）定义违反规则的自定义兜底逻辑，若不做自定义默认抛出异常

![image-20251207033432366](https://raw.githubusercontent.com/chen-1110/image/main/image-20251207033432366.png)

总体流程如上图，违反规则抛异常，若有自定义异常处理逻辑执行兜底逻辑，若无自定义逻辑默认抛异常。

### 不同资源的异常处理

![image-20251207033503253](https://raw.githubusercontent.com/chen-1110/image/main/image-20251207033503253.png)

违反规则默认抛异常，可以自定义异常处理逻辑，

- 对于web接口，可以实现自定义BlockExceptionHandler来自定义兜底逻辑

```
@Component
public class MyBlockExceptionHandler implements BlockExceptionHandler {
    private ObjectMapper objectMapper = new ObjectMapper();
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       String resourceName, BlockException e) throws Exception {
        response.setStatus(429); //too many requests
        response.setContentType("application/json;charset=utf-8");
        PrintWriter writer = response.getWriter();
        R error = R.error(500, resourceName + " 被Sentinel限制了，原因：" + e.getClass());
        String json = objectMapper.writeValueAsString(error);
        writer.write(json);
        writer.flush();
        writer.close();
    }
}
```

- 对于使用注解修饰的函数方法，可以创建兜底方法来自定义兜底逻辑（推荐使用）

```
    @SentinelResource(value = "createOrder", fallback = "createOrderFallback")
    public String createOrder(Long userId) {
        return productServiceFeignClient.getProductById(2222L);
    }

    public String createOrderFallback(Throwable e) {
        return "createOrderFallback";
    }
```

- 对于远程调用，可以创建fallback类自定义兜底逻辑

### 不同规则类型

#### 限流

推荐使用qps阈值（并发需要额外计算损耗）

分为集群和单机两种模式，集群模式下可以选择为集群设置总qps限流或单台机器限流

流控模式：最简单的是单资源限制，链路方式可以只对某一上游限制，关联较复杂略去。

流控效果：最简单是快速失败（推荐），warm up是处理类似预热场景，如设置qps100，10s，前10sqps限制会从较小值逐步放开到100，排队等待是限制qps数，其他线程等待。

![image-20251207034719315](https://raw.githubusercontent.com/chen-1110/image/main/image-20251207034719315.png)

#### 熔断

熔断规则可以设置为按慢调用比例/异常比例/异常数来熔断，要注意的是熔断规则是配置了规则前调用逻辑里错误直接执行fallback逻辑，配置后满足熔断**不调用**直接执行fallback逻辑

熔断规则的流程是，违反规则后熔断一定时长，该时长内不会发起调用直接执行兜底逻辑，时长过后会尝试发起一次调用若仍错误继续熔断。

![image-20251207040124680](https://raw.githubusercontent.com/chen-1110/image/main/image-20251207040124680.png)

#### 热点规则

热点规则较nb，可以根据方法的参数进行热点控制，可以有效避免redis热key等问题。

## gateway

http业务网关，提供了类似nginx的L6转发和负载均衡能力，相较nginx融合了spring生态，可以进行更细粒度的代码控制，可以实现鉴权等公共逻辑。

业务使用上，主要用途是写好路径匹配规则，将统一请求转发至各业务服务

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-route
          uri: lb://service-order
          predicates:
            - Path=/api/order/**
        - id: product-route
          uri: lb://service-product
          predicates:
            - Path=/api/product/**
          filters:
            - RewritePath=/api/product/?(?<segment>.*), /$\{segment}
```

## seata

分布式事务解决方案，一个@GlobalTransaction注解即可提供分布式事务能力。

### 分布式事务基础概念：

1.TC事务协调者，维护全局事务状态，控制各分支事务回滚逻辑

2.TM事务管理器，全局事务发起者

3.RM资源管理器，被TM引入的各分支事务

![image-20251207041313968](https://raw.githubusercontent.com/chen-1110/image/main/image-20251207041313968.png)

![image-20251207041322270](https://raw.githubusercontent.com/chen-1110/image/main/image-20251207041322270.png)

### seata简单底层实现

seata默认采用AT模式，底层上分支事务提交时，会记录修改到undolog里，同时会对分支事务修改的数据行加锁，TC若收到某个分支事务失败的信号，会通知各分支事务通过undolog回滚，若所有分支事务都成功，TC会通知各分支事务释放锁并清理undolog

![image-20251207041649641](https://raw.githubusercontent.com/chen-1110/image/main/image-20251207041649641.png)

此外seata还支持事务模式

XA：类似TA，区别是各分支事务不会提交，直到TC通知全部成功，缺陷很显然事务和锁时间过长

TCC: 业务需自定义实现分支事务的prepare，commit，rollback阶段，适合发邮件等广义事务场景，commit发邮件后可在rollback补发勘误邮件

Saga：通过消息队列实现最终一致性，适用于大事务场景。
