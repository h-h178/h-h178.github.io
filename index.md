# Docker



## 安装Docker

windows 安装 [Docker Desktop Installer](https://www.docker.com/) 选择 **Download for Windows - AMD64** 
并且windows需要开启 hyper-v 安装wsl 详见 [Windows安装使用Docker](https://blog.csdn.net/qq_60750453/article/details/128636298) 

配置阿里云镜像：

```
"registry-mirrors": ["https://zugnrk24.mirror.aliyuncs.com"]
```

## 常见命令

| **命令**       | **说明**                       | **文档地址**                                                 |
| :------------- | :----------------------------- | :----------------------------------------------------------- |
| docker pull    | 拉取镜像                       | [docker pull](https://docs.docker.com/engine/reference/commandline/pull/) |
| docker push    | 推送镜像到DockerRegistry       | [docker push](https://docs.docker.com/engine/reference/commandline/push/) |
| docker images  | 查看本地镜像                   | [docker images](https://docs.docker.com/engine/reference/commandline/images/) |
| docker rmi     | 删除本地镜像                   | [docker rmi](https://docs.docker.com/engine/reference/commandline/rmi/) |
| docker run     | 创建并运行容器（不能重复创建） | [docker run](https://docs.docker.com/engine/reference/commandline/run/) |
| docker stop    | 停止指定容器                   | [docker stop](https://docs.docker.com/engine/reference/commandline/stop/) |
| docker start   | 启动指定容器                   | [docker start](https://docs.docker.com/engine/reference/commandline/start/) |
| docker restart | 重新启动容器                   | [docker restart](https://docs.docker.com/engine/reference/commandline/restart/) |
| docker rm      | 删除指定容器                   | [docs.docker.com](https://docs.docker.com/engine/reference/commandline/rm/) |
| docker ps      | 查看容器                       | [docker ps](https://docs.docker.com/engine/reference/commandline/ps/) |
| docker logs    | 查看容器运行日志               | [docker logs](https://docs.docker.com/engine/reference/commandline/logs/) |
| docker exec    | 进入容器                       | [docker exec](https://docs.docker.com/engine/reference/commandline/exec/) |
| docker save    | 保存镜像到本地压缩文件         | [docker save](https://docs.docker.com/engine/reference/commandline/save/) |
| docker load    | 加载本地压缩文件到镜像         | [docker load](https://docs.docker.com/engine/reference/commandline/load/) |
| docker inspect | 查看容器详细信息               | [docker inspect](https://docs.docker.com/engine/reference/commandline/inspect/) |



# 远程调用(可不掌握)

Spring给我们提供了一个RestTemplate的API，可以方便的实现Http请求的发送。

```java
@Configuration
public class RemoteCallConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```java
ResponseEntity<List<ItemDTO>> response = restTemplate.exchange(
            "http://localhost:8081/items?ids={ids}",
            HttpMethod.GET,
            null,
            new ParameterizedTypeReference<List<ItemDTO>>() {
            },
            Map.of("ids", CollUtil.join(itemIds, ","))
    );
```



# nacos注册中心

注册中心有许多

目前开源的注册中心框架有很多，国内比较常见的有：

- Eureka：Netflix公司出品，目前被集成在SpringCloud当中，一般用于Java应用
- Nacos：Alibaba公司出品，目前被集成在SpringCloudAlibaba中，一般用于Java应用
- Consul：HashiCorp公司出品，目前集成在SpringCloud中，不限制微服务语言

 流程如下：

- 服务启动时就会注册自己的服务信息（服务名、IP、端口）到注册中心
- 调用者可以从注册中心订阅想要的服务，获取服务对应的实例列表（1个服务可能多实例部署）
- 调用者自己对实例列表负载均衡，挑选一个实例
- 调用者向该实例发起远程调用

当服务提供者的实例宕机或者启动新实例时，调用者如何得知呢？

- 服务提供者会定期向注册中心发送请求，报告自己的健康状态（心跳请求）
- 当注册中心长时间收不到提供者的心跳时，会认为该实例宕机，将其从服务的实例列表中剔除
- 当服务有新实例启动时，会发送注册服务请求，其信息会被记录在注册中心的服务实例列表
- 当注册中心服务列表变更时，会主动通知微服务，更新本地服务列表

## Docker部署nacos

env：

```properties
PREFER_HOST_MODE=hostname
MODE=standalone
SPRING_DATASOURCE_PLATFORM=mysql
MYSQL_SERVICE_HOST=192.168.83.101
MYSQL_SERVICE_DB_NAME=nacos
MYSQL_SERVICE_PORT=3307
MYSQL_SERVICE_USER=root
MYSQL_SERVICE_PASSWORD=123456
MYSQL_SERVICE_DB_PARAM=characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Shanghai
```

安装镜像：

```
docker load -i "D:/BaiduNetdiskDownload/nacos.tar"
```

引用上面的env,配置数据挂载在本机目录下

```
docker run -d --name nacos --env-file "E:/Docker/nacos/custom.env" -p 8848:8848 -p 9848:9848 -p 9849:9849 --restart=always -v "E:/Docker/nacos/data:/home/nacos/data" -v "E:/Docker/nacos/logs:/home/nacos/logs" nacos/nacos-server:v2.1.0-slim
```

访问nacos：

http://192.168.83.101:8848/nacos 使用自己网络地址

## 引入依赖

```xml
<!--nacos 服务注册发现-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

## 配置nacos

`application.yml`中添加nacos地址配置：

```yaml
spring:
  application:
    name: item-service # 服务名称
  cloud:
    nacos:
      server-addr: 192.168.83.101:8848 # nacos地址
```

## 服务发现

手写服务发现，随机数获取当前服务实例

```java
    private final DiscoveryClient discoveryClient;
    // 根据服务名称获取服务的实例列表
    List<ServiceInstance> instances = discoveryClient.getInstances("item-service");
    // 手写负载均衡，从实例列表挑选一个实例
    ServiceInstance instance = instances.get(RandomUtil.randomInt(instances.size()));
	// 获取实例的IP和接口
	URI uri = instance.getUri();
```

当然，常见的负载均衡算法有：

- 随机
- 轮询
- IP的hash
- 最近最少访问

# OpenFeigh

OpenFeign是一个声明式的http客户端，是SpringCloud在Eureka公司开源的Feign基础上改造而来。官方地址：[OpenFeign](https//github.com/OpenFeign/feign) 其作用就是基于SpringMVC的常见注解，帮我们优雅的视线http请求的发送。

## OpenFeign的使用

### 引入依赖

```xml
        <!--openFeign-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <!--负载均衡器-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </dependency>
```

### 启用OpenFeign功能

通过@EnableFeignClients注解，启用

```java
@EnableFeignClients
@SpringBootApplication
public class CartApplication { // ... 略}
```

### 编写 FeignClient 接口

```java
@FeignClient(value = "item-service")
public interface ItemClient {

    @GetMapping("/items")
    List<ItemDTO> queryItemByIds(@RequestParam("ids") Collection<Long> ids);
}
```

### 调用 FeignClient 接口

```java
    private final ItemClient itemClient;
    
	List<ItemDTO> items = itemClient.queryItemByIds(itemIds);
```

## 连接池

OpenFeign默认每次发起请求都会创建连接、销毁连接

Feign底层发起http请求，依赖于其它的框架。其底层支持的http客户端实现包括：

- HttpURLConnection：默认实现，不支持连接池
- Apache HttpClient ：支持连接池
- OKHttp：支持连接池

因此我们通常会使用带有连接池的客户端来代替默认的HttpURLConnection。比如，我们使用OK Http。

### OpenFeign整合OKHttp

1. ###### 引入依赖

   ```xml
   <!--OK http 的依赖 -->
   <dependency>
     <groupId>io.github.openfeign</groupId>
     <artifactId>feign-okhttp</artifactId>
   </dependency>
   ```

2. 开启连接池

   ```yaml
   feign:
     okhttp:
       enabled: true # 开启OKHttp功能
   ```

## 最佳实践

创建新模块hm-api 用来统一处理

### 引入依赖

```xml
<dependencies>
        <!--open feign-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <!-- load balancer-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </dependency>
        <!-- swagger 注解依赖 -->
        <dependency>
            <groupId>io.swagger</groupId>
            <artifactId>swagger-annotations</artifactId>
            <version>1.6.6</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>
```

创建client、dto包

### 引入hm-api模块

```xml
  <!--feign模块-->
  <dependency>
      <groupId>com.heima</groupId>
      <artifactId>hm-api</artifactId>
      <version>1.0.0</version>
  </dependency>
```

### 声明扫描包

让引入的服务下Springboot扫描到包，自动引入bean

```java
@EnableFeignClients(basePackages = "com.hmall.api.client")
```

## 日志配置

OpenFeign只会在FeignClient所在包的日志级别为**DEBUG**时，才会输出日志。而且其日志级别有4级：

- **NONE**：不记录任何日志信息，这是默认值。
- **BASIC**：仅记录请求的方法，URL以及响应状态码和执行时间
- **HEADERS**：在BASIC的基础上，额外记录了请求和响应的头信息
- **FULL**：记录所有请求和响应的明细，包括头信息、请求体、元数据。

Feign默认的日志级别就是NONE，所以默认我们看不到请求日志。

### 定义日志级别

在hm-api模块下新建一个配置类，定义Feign的日志级别：

```java
public class DefaultFeignConfig {
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

### 配置

1. 全局配置

   ```java
   @EnableFeignClients(defaultConfiguration = DefaultFeignConfig.class)
   ```

2. 局部配置

   ```java
   @FeignClient(value = "item-service", configuration = DefaultFeignConfig.class)
   ```

日志格式：

```
23:13:59:329 DEBUG 21172 --- [nio-8082-exec-1] c.h.cart.mapper.CartMapper.selectList    : ==>  Preparing: SELECT id,user_id,item_id,num,name,spec,price,image,create_time,update_time FROM cart WHERE (user_id = ?)
23:13:59:352 DEBUG 21172 --- [nio-8082-exec-1] c.h.cart.mapper.CartMapper.selectList    : ==> Parameters: 1(Long)
23:13:59:378 DEBUG 21172 --- [nio-8082-exec-1] c.h.cart.mapper.CartMapper.selectList    : <==      Total: 1
23:13:59:438 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] ---> GET http://item-service/items?ids=100000006163 HTTP/1.1
23:13:59:438 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] ---> END HTTP (0-byte body)
23:13:59:696 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] <--- HTTP/1.1 200  (257ms)
23:13:59:696 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] connection: keep-alive
23:13:59:696 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] content-type: application/json
23:13:59:696 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] date: Thu, 26 Sep 2024 15:13:59 GMT
23:13:59:697 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] keep-alive: timeout=60
23:13:59:697 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] transfer-encoding: chunked
23:13:59:697 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] 
23:13:59:698 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] [{"id":"100000006163","name":"巴布豆(BOBDOG)柔薄悦动婴儿拉拉裤XXL码80片(15kg以上)","price":67100,"stock":10000,"image":"https://m.360buyimg.com/mobilecms/s720x720_jfs/t23998/350/2363990466/222391/a6e9581d/5b7cba5bN0c18fb4f.jpg!q70.jpg.webp","category":"拉拉裤","brand":"巴布豆","spec":"{}","sold":11,"commentCount":33343434,"isAD":false,"status":2}]
23:13:59:698 DEBUG 21172 --- [nio-8082-exec-1] com.hmall.api.client.ItemClient          : [ItemClient#queryItemByIds] <--- END HTTP (371-byte body)
```

# Gateway网关

**网关路由**对应的java类型是RouteDefinition，其中常见的属性有：

- `id`：路由的唯一标示
- `predicates`：路由断言，判断请求是否符合当前路由
- `filters`：路由过滤器，对请求或者响应做特殊处理
- `uri`：路由目标地址，`lb://`代表负载均衡，从注册中心获取目标微服务的实例列表，并且负载均衡选择一个访问。

## 启动

### 引入依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>hmall</artifactId>
        <groupId>com.heima</groupId>
        <version>1.0.0</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>hm-gateway</artifactId>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
    </properties>
    <dependencies>
        <!--common-->
        <dependency>
            <groupId>com.heima</groupId>
            <artifactId>hm-common</artifactId>
            <version>1.0.0</version>
        </dependency>
        <!--网关-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <!--nacos discovery-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <!--负载均衡-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </dependency>
    </dependencies>
    <build>
        <finalName>${project.artifactId}</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

###  启动类

```java
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

### 配置路由规则

```yaml
server:
  port: 8080
spring:
  application:
    name: gateway
  cloud:
    nacos:
      server-addr: 192.168.83.103:8848
    gateway:
      routes:
        # 商品
        - id: item-service # 路由规则id，自定义，唯一
          uri: lb://item-service # 路由的目标服务，lb代表负载均衡，会从注册中心拉取服务列表
          predicates: # 路由断言，判断当前请求是否符合当前规则，符合则路由到目标服务
            - Path=/items/**, /search/** # 这里是以请求路径作为判断规则
        # 用户
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/addresses/**, /users/**
        # 交易
        - id: trade-service
          uri: lb://trade-service
          predicates:
            - Path=/orders/**
        # 支付
        - id: pay-service
          uri: lb://pay-service
          predicates:
            - Path=/pay-orders/**

```

### 路由断言

| **名称**   | **说明**                       | **示例**                                                     |
| :--------- | :----------------------------- | :----------------------------------------------------------- |
| After      | 是某个时间点后的请求           | - After=2037-01-20T17:42:47.789-07:00[America/Denver]        |
| Before     | 是某个时间点之前的请求         | - Before=2031-04-13T15:14:47.433+08:00[Asia/Shanghai]        |
| Between    | 是某两个时间点之前的请求       | - Between=2037-01-20T17:42:47.789-07:00[America/Denver], 2037-01-21T17:42:47.789-07:00[America/Denver] |
| Cookie     | 请求必须包含某些cookie         | - Cookie=chocolate, ch.p                                     |
| Header     | 请求必须包含某些header         | - Header=X-Request-Id, \d+                                   |
| Host       | 请求必须是访问某个host（域名） | - Host=**.somehost.org,**.anotherhost.org                    |
| Method     | 请求方式必须是指定方式         | - Method=GET,POST                                            |
| Path       | 请求路径必须符合指定规则       | - Path=/red/{segment},/blue/**                               |
| Query      | 请求参数必须包含指定参数       | - Query=name, Jack或者- Query=name                           |
| RemoteAddr | 请求者的ip必须是指定范围       | - RemoteAddr=192.168.1.1/24                                  |
| weight     | 权重处理                       |                                                              |

### 路由过滤

1. 单个服务配置

   ```yaml
   server:
     port: 8080
   spring:
     application:
       name: gateway
     cloud:
       nacos:
         server-addr: 192.168.83.103:8848
       gateway:
         routes:
           # 商品
           - id: item-service
             uri: lb://item-service
             predicates:
               - Path=/items/**, /search/**
             filters:
               - AddRequestHeader=truth, anyone long-press like button will be rich # key,value
   ```

2. 全局配置

   ```yaml
   server:
     port: 8080
   spring:
     application:
       name: gateway
     cloud:
       nacos:
         server-addr: 192.168.83.103:8848
       gateway:
         routes: <4 items>
         default-filters:
           - AddRequestHeader=truth, anyone long-press like button will be rich # key,value
   
   ```

## 	网关登录校验

1. 客户端请求进入网关后由`HandlerMapping`对请求做判断，找到与当前请求匹配的路由规则（**`Route`**），然后将请求交给`WebHandler`去处理。
2. `WebHandler`则会加载当前路由下需要执行的过滤器链（**`Filter chain`**），然后按照顺序逐一执行过滤器（后面称为**`Filter`**）。
3. 每个`Filter`内部的逻辑分为`pre`和`post`两部分，`pre` 逻辑会在请求被路由到微服务之前执行，`post` 逻辑则在微服务返回结果之后执行。
4. 只有所有`Filter`的`pre`逻辑都依次顺序执行通过后，请求才会被路由到微服务。
5. 微服务返回结果后，再倒序执行`Filter`的`post`逻辑。
6. 最终把响应结果返回。

最终请求转发是有一个名为`NettyRoutingFilter`的过滤器来执行的，而且这个过滤器是整个过滤器链中顺序最靠后的一个。**如果我们能够定义一个过滤器，在其中实现登录校验逻辑，并且将过滤器执行顺序定义到**`NettyRoutingFilter`**之前**，这就符合我们的需求了！

### 自定义过滤器

**网关过滤器链**中的过滤器有两种：

- **`GatewayFilter`**：路由过滤器，作用范围比较灵活，可以是任意指定的路由`Route`，默认不生效，要配置路由后生效。 
- **`GlobalFilter`**：全局过滤器，作用范围是所有路由，不可配置，声明后自动生效。

`GatewayFilter`和`GlobalFilter`这两种过滤器的方法签名完全一致：

```Java
/**
 * 处理请求并将其传递给下一个过滤器
 * @param exchange 当前请求的上下文，其中包含request、response等各种数据
 * @param chain 过滤器链，基于它向下传递请求
 * @return 根据返回值标记当前请求是否被完成或拦截，chain.filter(exchange)就放行了。
 */
Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
```

- `ServerWebExchange exchange`：请求上下文，包含整个过滤器链内共享数据，例如request、response等
- `GatewayFilterChain chain`：过滤器链，当前过滤器执行完毕后，要调用过滤器链中的下一个过滤器

#### 自定义GlobalFilter

自定义`GlobalFilter`，实现`GlobalFilter`接口:

```java
@Component
public class MyGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // TODO 模拟登录校验逻辑
        ServerHttpRequest request = exchange.getRequest();
        HttpHeaders headers = request.getHeaders();
        System.out.println("headers=" + headers);
        // 放行
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
    	// 过滤器执行顺序，值越小，优先级越高 
        return 0;
    }
}
```

#### 自定义GatewayFilter

自定义`GatewayFilter`不是直接实现`GatewayFilter`而是实现`AbstractGatewayFilterFactory`:

- 无参

  ```java
  @Component
  public class PrintAnyGatewayFilterFactory extends AbstractGatewayFilterFactory {
      @Override
      public GatewayFilter apply(Object config) {
      // OrderedGatewayFilter 装饰类
          return new OrderedGatewayFilter(new GatewayFilter() {
              @Override
              public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
                  // 业务代码
                  System.out.println("print any filter running");
                  // 放行
                  return chain.filter(exchange);
              }
          }, 1);
      }
  }
  ```

  **注意**：该类的名称一定要以`GatewayFilterFactory`为后缀！

  然后在yaml配置中这样使用：

  ```YAML
  spring:
    cloud:
      gateway:
        default-filters:
              - PrintAny # 此处直接以自定义的GatewayFilterFactory类名称前缀类声明过滤器
  ```

- 有参

  ```java
  @Component
  public class PrintAnyGatewayFilterFactory extends AbstractGatewayFilterFactory<PrintAnyGatewayFilterFactory.Config> {
      @Override
      public GatewayFilter apply(Config config) {
      // OrderedGatewayFilter 装饰类
          return new OrderedGatewayFilter(new GatewayFilter() {
              @Override
              public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
                  // 业务代码
                  String a = config.getA();
                  String b = config.getB();
                  String c = config.getC();
                  System.out.println("a=" + a);
                  System.out.println("b=" + b);
                  System.out.println("c=" + c);
                  System.out.println("print any filter running");
                  // 放行
                  return chain.filter(exchange);
              }
          }, 1);
      }
  
      // 自定义配置属性，成员变量名称很重要，下面会用到
      @Data
      public static class Config{
          private String a;
          private String b;
          private String c;
      }
      // 将变量名称依次返回，顺序很重要，将来读取参数时需要按顺序获取
      @Override
      public List<String> shortcutFieldOrder() {
          return List.of("a", "b", "c");
      }
      // 将Config字节码传递给父类，父类负责帮我们读取yaml配置
      public PrintAnyGatewayFilterFactory () {
          super(Config.class);
      }
  }
  ```

  yaml配置：

  ```YAML
  spring:
    cloud:
      gateway:
        default-filters:
              - PrintAny=1,2,3 # 注意，这里多个参数以","隔开，将来会按照shortcutFieldOrder()方法返回的参数顺序依次复制
  ```

  上面这种配置方式参数必须严格按照shortcutFieldOrder()方法的返回参数名顺序来赋值。

  还有一种用法，无需按照这个顺序，就是手动指定参数名：

  ```YAML
  spring:
    cloud:
      gateway:
        default-filters:
              - name: PrintAny
                args: # 手动指定参数名，无需按照参数顺序
                  a: 1
                  b: 2
                  c: 3
  ```

### 登录效验

具体作用如下：

- `AuthProperties`：配置登录校验需要拦截的路径，因为不是所有的路径都需要登录才能访问
- `JwtProperties`：定义与JWT工具有关的属性，比如秘钥文件位置
- `SecurityConfig`：工具的自动装配
- `JwtTool`：JWT工具，其中包含了校验和解析`token`的功能
- `hmall.jks`：秘钥文件

其中`AuthProperties`和`JwtProperties`所需的属性要在`application.yaml`中配置：

```YAML
hm:
  jwt:
    location: classpath:hmall.jks # 秘钥地址
    alias: hmall # 秘钥别名
    password: hmall123 # 秘钥文件密码
    tokenTTL: 30m # 登录有效期
  auth:
    excludePaths: # 无需登录校验的路径
      - /search/**
      - /users/login
      - /items/**
```

**登录效验过滤器**

```java
@Component
@RequiredArgsConstructor
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    private final AuthProperties authProperties;

    private final JwtTool jwtTool;

    private final AntPathMatcher antPathMatcher = new AntPathMatcher();

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 1.获取request
        ServerHttpRequest request = exchange.getRequest();
        // 2.判断是否需要做拦截
        if (isExlude(request.getPath().toString())) {
            // 放行
            return chain.filter(exchange);
        }
        // 3.获取token
        String token = null;
        List<String> headers = request.getHeaders().get("authorization");
        if (headers != null && !headers.isEmpty()) {
            token = headers.get(0);
        }
        // 4.效验并解析token
        Long userId = null;
        try {
            userId = jwtTool.parseToken(token);
        } catch (UnauthorizedException e) {
            // 拦截，设置响应码401
            ServerHttpResponse response = exchange.getResponse();
            response.setStatusCode(HttpStatus.UNAUTHORIZED);
            return response.setComplete();
        }
        // TODO 5.传递用户信息
        System.out.println("userId = " + userId);

        // 6.放行
        return chain.filter(exchange);
    }

    // 放行路径效验
    private boolean isExlude(String path) {
        for (String pathPatter: authProperties.getExcludePaths()) {
            if (antPathMatcher.match(pathPatter, path)) {
                return true;
            }
        }
        return false;
    }

    @Override
    public int getOrder() {
        return 0;
    }
}

```

### 网关传递用户

改造上面的登录效验

```java
ServerWebExchange swe = exchange.mutate()
    .request(builder -> builder.header("user-info", userInfo))
    .build();
```

#### 注册拦截器

```java
public class UserInfoInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 1.获取登录信息
        String userInfo = request.getHeader("user-info");
        // 2.判断是否获取了用户，如果有。存入ThreadLocal
        if (StrUtil.isNotBlank(userInfo)) {
            UserContext.setUser(Long.valueOf(userInfo));
        }
        // 3.放行

        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        // 清理用户
        UserContext.removeUser();
    }
}

```

#### MVC自动装配拦截器

```java
@Configuration
@ConditionalOnClass(DispatcherServlet.class)
public class MvcConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new UserInfoInterceptor());
    }
}

```

不过，需要注意的是，这个配置类默认是不会生效的，因为它所在的包是`com.hmall.common.config`，与其它微服务的扫描包不一致，无法被扫描到，因此无法生效。

基于SpringBoot的自动装配原理，我们要将其添加到`resources`目录下的`META-INF/spring.factories`文件中：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.hmall.common.config.MyBatisConfig,\
  com.hmall.common.config.MvcConfig
```

**注意：**因为`hm-gateway`模块下没用引用SpringMVC依赖,但`hm-gateway`引用了`hm-common`模块，需要在`MvcConfig`类上配置条件判断

```java
@ConditionalOnClass(DispatcherServlet.class)
```

`@ConditionalOnClass(DispatcherServlet.class)` 是一个 Spring Boot 注解，用于在指定的类（例如 `DispatcherServlet`）存在于类路径中时，才有条件地启用某个配置 bean。

`DispatcherServlet` 是 Spring MVC 的核心类，负责处理 HTTP 请求。这个注解通常用于在 Web 环境和 Spring MVC 可用时才生效的配置中。

### OpenFeign传递用户

OpenFeign中提供的一个拦截器接口：`feign.RequestInterceptor`

```Java
public interface RequestInterceptor {

  /**
   * Called for every request. 
   * Add data using methods on the supplied {@link RequestTemplate}.
   */
  void apply(RequestTemplate template);
}
```

**实现**

```java
@Bean
public RequestInterceptor userInfoRequestInterceptor() {
    return new RequestInterceptor() {
        @Override
        public void apply(RequestTemplate template) {
            Long userId = UserContext.getUser();
            if (userId != null) {
                template.header("user-info", userId.toString());
            }
        }
    };
}
```

OpenFeign需要调用`UserContext.getUser()`也需要引入hm-common依赖

```xml
    <!--common-->
    <dependency>
        <groupId>com.heima</groupId>
        <artifactId>hm-common</artifactId>
        <version>1.0.0</version>
    </dependency>
```

# 配置管理



## 配置共享



### 添加配置共享

**mysql:**

```yaml
spring:
  datasource:
    url: jdbc:mysql://${hm.db.host:192.168.83.103}:{hm.db.port:3307}/{hm.db.database}?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: {hm.db.un:root}
    password: ${hm.db.pw:123456}
mybatis-plus:
  configuration:
    default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
  global-config:
    db-config:
      update-strategy: not_null
      id-type: auto
```

**log：**

```yaml
logging:
  level:
    com.hmall: debug
  pattern:
    dateformat: HH:mm:ss:SSS
  file:
    path: "logs/${spring.application.name}"
```

**swagger：**

```yaml
knife4j:
  enable: true
  openapi:
    title: ${hm.swagger.title:黑马商城接口文档}
    description: ${hm.swagger.description:黑马商城接口文档}
    email: zhanghuyi@itcast.cn
    concat: 虎哥
    url: https://www.itcast.cn
    version: v1.0.0
    group:
      default:
        group-name: default
        api-rule: package
        api-rule-resources:
          - ${hm.swagger.package}
```

### 拉取配置共享

**引入依赖**

```xml
  <!--nacos配置管理-->
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
  </dependency>
  <!--读取bootstrap文件-->
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-bootstrap</artifactId>
  </dependency>
```

**创建bootstrap.yaml**



```yaml
server:
  port: 8082
feign:
  okhttp:
    enabled: true # 开启OKHttp功能
hm:
  db:
    database: hm-cart
  swagger:
    title: "黑马商城购物车服务接口文档"
    package: com.hmall.cart.controller
```

## 配置热更新

注意文件的dataId格式：

```Plain
[服务名]-[spring.active.profile].[后缀名]
```

文件名称由三部分组成：

- **`服务名`**：我们是购物车服务，所以是`cart-service`
- **`spring.active.profile`**：就是spring boot中的`spring.active.profile`，可以省略，则所有profile共享该配置
- **`后缀名`**：例如yaml

### nacos配置

```yaml
hm:
  cart:
    maxAmount: 1 # 购物车商品数量上限
```

### 配置热更新文件

```java
@Data
@Component
@ConfigurationProperties(prefix = "hm.cart")
public class CartProperties {
    private Integer maxItems;
}
```

直接注入调用属性

## 动态路由

网关的路由配置全部是在项目启动时由`org.springframework.cloud.gateway.route.CompositeRouteDefinitionLocator`在项目启动的时候加载，并且一经加载就会缓存到内存中的路由表内（一个Map），不会改变。也不会监听路由变更，所以，我们无法利用上节课学习的配置热更新来实现路由更新。

### 监听Nacos配置变更

在Nacos官网中给出了手动监听Nacos配置变更的SDK：

https://nacos.io/zh-cn/docs/sdk.html

如果希望 Nacos 推送配置变更，可以使用 Nacos 动态监听配置接口来实现。

```Java
public void addListener(String dataId, String group, Listener listener)
```

请求参数说明：

| **参数名** | **参数类型** | **描述**                                                     |
| :--------- | :----------- | :----------------------------------------------------------- |
| dataId     | string       | 配置 ID，保证全局唯一性，只允许英文字符和 4 种特殊字符（"."、":"、"-"、"_"）。不超过 256 字节。 |
| group      | string       | 配置分组，一般是默认的DEFAULT_GROUP。                        |
| listener   | Listener     | 监听器，配置变更进入监听器的回调函数。                       |

### ConfigService

```java
String getConfigAndSignListener(
    String dataId, // 配置文件id
    String group, // 配置组，走默认
    long timeoutMs, // 读取配置的超时时间
    Listener listener // 监听器
) throws NacosException;
```

### 更新路由

```Java
package org.springframework.cloud.gateway.route;

import reactor.core.publisher.Mono;

/**
 * @author Spencer Gibb
 */
public interface RouteDefinitionWriter {
        /**
     * 更新路由到路由表，如果路由id重复，则会覆盖旧的路由
     */
        Mono<Void> save(Mono<RouteDefinition> route);
        /**
     * 根据路由id删除某个路由
     */
        Mono<Void> delete(Mono<String> routeId);

}
```

其中`RouteDefinition`就是路由表，一般写成JSON格式，格式如下：

```json
{
  "id": "item",
  "predicates": [{
    "name": "Path",
    "args": {"_genkey_0":"/items/**", "_genkey_1":"/search/**"}
  }],
  "filters": [],
  "uri": "lb://item-service"
}
```

等同于

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: item
          uri: lb://item-service
          predicates:
            - Path=/items/**,/search/**
```

### 实现动态路由

#### 引入依赖

```xml
<!--统一配置管理-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<!--加载bootstrap-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

#### 修改配置文件

然后在网关`gateway`的`resources`目录创建`bootstrap.yaml`文件，内容如下：

```YAML
spring:
  application:
    name: gateway
  cloud:
    nacos:
      server-addr: 192.168.1.103
      config:
        file-extension: yaml
        shared-configs:
          - dataId: shared-log.yaml # 共享日志配置
```

接着，修改`gateway`的`resources`目录下的`application.yml`，把之前的路由移除，最终内容如下：

```YAML
server:
  port: 8080 # 端口
hm:
  jwt:
    location: classpath:hmall.jks # 秘钥地址
    alias: hmall # 秘钥别名
    password: hmall123 # 秘钥文件密码
    tokenTTL: 30m # 登录有效期
  auth:
    excludePaths: # 无需登录校验的路径
      - /search/**
      - /users/login
      - /items/**
```

#### 配置监听器

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class DynamicRouteLoader {
    private final NacosConfigManager nacosConfigManager;
    private final RouteDefinitionWriter write;
    private final String dataId = "gateway-routes.json";
    private final String group = "DEFAULT_GROUP";
    private final Set<String> routeIds = new HashSet<>();

    @PostConstruct
    public void initRouteConfigListener() throws NacosException {
        // 1.项目启动时，先拉取一次配置，并且添加配置监听器
        String configInfo = nacosConfigManager.getConfigService()
                .getConfigAndSignListener(dataId, group, 5000, new Listener() {
                    @Override
                    public Executor getExecutor() {
                        return null;
                    }
                    @Override
                    public void receiveConfigInfo(String configInfo)  {
                        // 2.监听到配置变更，需要去更新路由表
                        updateConfigInfo(configInfo);
                    }
                });
        // 3.第一次读取到配置，也需要更新到路由表
        updateConfigInfo(configInfo);
    }

    public void updateConfigInfo(String configInfo)  {
        log.debug("监听到路由信息：{}", configInfo);
        // 1.解析配置文件，转为RouteDefinition
        List<RouteDefinition> routeDefinitions = JSONUtil.toList(configInfo, RouteDefinition.class);
        // 2.删除旧的路由表
        for (String id : routeIds)  {
            write.delete(Mono.just(id)).subscribe();
        }
        routeIds.clear();
        // 3.更新路由表
        for (RouteDefinition routeDefinition : routeDefinitions)  {
            // 3.1.更新路由表
            write.save(Mono.just(routeDefinition)).subscribe();
            // 3.2.记录路由id，便于下次删除
            routeIds.add(routeDefinition.getId());
        }
    }
}
```

nacos中添加路由`gateway-routes.json`，类型为`json`：

```json
[
    {
        "id": "item",
        "predicates": [{
            "name": "Path",
            "args": {"_genkey_0":"/items/**", "_genkey_1":"/search/**"}
        }],
        "filters": [],
        "uri": "lb://item-service"
    },
    {
        "id": "cart",
        "predicates": [{
            "name": "Path",
            "args": {"_genkey_0":"/carts/**"}
        }],
        "filters": [],
        "uri": "lb://cart-service"
    },
    {
        "id": "user",
        "predicates": [{
            "name": "Path",
            "args": {"_genkey_0":"/users/**", "_genkey_1":"/addresses/**"}
        }],
        "filters": [],
        "uri": "lb://user-service"
    },
    {
        "id": "trade",
        "predicates": [{
            "name": "Path",
            "args": {"_genkey_0":"/orders/**"}
        }],
        "filters": [],
        "uri": "lb://trade-service"
    },
    {
        "id": "pay",
        "predicates": [{
            "name": "Path",
            "args": {"_genkey_0":"/pay-orders/**"}
        }],
        "filters": [],
        "uri": "lb://pay-service"
    }
]
```

# Sentinel

**下载安装**

Sentinel是阿里巴巴开源的一款服务保护框架，目前已经加入SpringCloudAlibaba中。官方网站：

https://sentinelguard.io/zh-cn/

Sentinel 的使用可以分为两个部分:

- **核心库**（Jar包）：不依赖任何框架/库，能够运行于 Java 8 及以上的版本的运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持。在项目中引入依赖即可实现服务限流、隔离、熔断等功能。
- **控制台**（Dashboard）：Dashboard 主要负责管理推送规则、监控、管理机器信息等。

为了方便监控微服务，我们先把Sentinel的控制台搭建出来。

1）下载jar包

下载地址：

https://github.com/alibaba/Sentinel/releases

2）运行

将jar包放在任意非中文、不包含特殊字符的目录下，重命名为`sentinel-dashboard.jar`：

然后运行如下命令启动控制台：

```Shell
java -Dserver.port=8090 -Dcsp.sentinel.dashboard.server=localhost:8090 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.6.jar
```

其它启动时可配置参数可参考官方文档：

https://github.com/alibaba/Sentinel/wiki/%E5%90%AF%E5%8A%A8%E9%85%8D%E7%BD%AE%E9%A1%B9

**微服务整合**

1）引入sentinel依赖

```XML
<!--sentinel-->
<dependency>
    <groupId>com.alibaba.cloud</groupId> 
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

2）配置控制台

修改application.yaml文件，添加下面内容：

```YAML
spring:
  cloud: 
    sentinel:
      transport:
        dashboard: localhost:8090 # sentinel 控制台地址
      http-method-specify: true # 开启对http method的限流，适用于Restful风格
```

**簇点链路**，就是单机调用链路，是一次请求进入服务后经过的每一个被`Sentinel`监控的资源。默认情况下，`Sentinel`会监控`SpringMVC`的每一个`Endpoint`（接口）。即，就是controller的每一个接口。

## 请求限流

在簇点链路后面点击流控按钮，即可对其做限流配置：

![img](assets/829a98661df5cf64b2d0d964cda61b3.png)

## 线程隔离

需要注意的是，默认情况下SpringBoot项目的tomcat最大线程数是200，允许的最大连接是8492，单机测试很难打满。

所以我们需要配置一下cart-service模块的application.yml文件，修改tomcat连接：

```YAML
server:
  port: 8082
  tomcat:
    threads:
      max: 50 # 允许的最大线程数
    accept-count: 50 # 最大排队等待数量
    max-connections: 100 # 允许的最大连接
```

![img](assets/55accea19f42b3bdd8995933d4ee2ff.png)

注意，这里勾选的是并发线程数限制，也就是说这个查询功能最多使用5个线程，而不是5QPS。如果查询商品的接口每秒处理2个请求，则5个线程的实际QPS在10左右，而超出的请求自然会被拒绝。

## 服务熔断

### Fallback

**启用OpenFeign的sentinel**

```yaml
feign:
  okhttp:
    enabled: true # 开启OKHttp功能
  sentinel:
    enabled: true # 开启Sentinel功能
```

**自定义类，实现`FallbackFactory`**

```java
@Slf4j
public class ItemClientFallbackFactory implements FallbackFactory<ItemClient> {
    @Override
    public ItemClient create(Throwable cause) {
        return new ItemClient() {
            @Override
            public List<ItemDTO> queryItemByIds(Collection<Long> ids) {
                // 记录失败日志
                log.error("远程调用ItemClient#queryItemByIds方法出现异常，参数：{}", ids, cause);
                // 查询购物车允许失败，查询失败，返回空集合
                return CollUtils.emptyList();
            }
            @Override
            public void deductStock(List<OrderDetailDTO> items) {
                // 库存扣减业务需要触发事务回滚，查询失败，抛出异常
                throw new BizIllegalException(cause);
            }
        };
    }
}
```

**在`DefaultFeignConfig`中注册为bean**

```java
    @Bean
    public ItemClientFallbackFactory itemClientFallbackFactory() {
        return new ItemClientFallbackFactory();
    }
```

**在`ItemClient`中引入：**

```java
@FeignClient(value = "item-service", fallbackFactory = ItemClientFallbackFactory.class)
```

注意：使用的服务中必须引入`defaultConfiguration = DefaultFeignConfig.class`

### 熔断

第一，超出的QPS上限的请求就只能抛出异常，从而导致购物车的查询失败。但从业务角度来说，即便没有查询到最新的商品信息，购物车也应该展示给用户，用户体验更好。也就是给查询失败设置一个**降级处理**逻辑。

第二，由于查询商品的延迟较高（模拟的500ms），从而导致查询购物车的响应时间也变的很长。这样不仅拖慢了购物车服务，消耗了购物车服务的更多资源，而且用户体验也很差。对于商品服务这种不太健康的接口，我们应该直接停止调用，直接走降级逻辑，避免影响到当前服务。也就是将商品查询接口**熔断**。

# Seata

在Seata的事务管理中有三个重要的角色：

-  **TC (Transaction Coordinator) - 事务协调者：**维护全局和分支事务的状态，协调全局事务提交或回滚。 
-  **TM (Transaction Manager) -** **事务管理器：**定义全局事务的范围、开始全局事务、提交或回滚全局事务。 
-  **RM (Resource Manager) -** **资源管理器：**管理分支事务，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。 

## Docker部署

**安装镜像**

```shell
docker load -i "E:\wfw\seata-1.5.2.tar"
```

**部署同一网络**

```shell
docker network connect [网络名] [容器名]
```

**创建运行容器**

```shell
docker run --name seata -p 8099:8099 -p 7099:7099 -e SEATA_IP=192.168.83.103 -v "E:/wfw/seata:/seata-server/resources" --privileged=true --network hm-net -d seataio/seata-server:1.5.2
```

**注意：**目前遇到填写容器名启动报错的原因，于是换成了容器内网内ip 自定义的同一network下的ip

## 微服务集成seata

为了方便各个微服务集成seata，我们需要把seata配置共享到nacos，因此模块不仅仅要引入seata依赖，还要引入nacos依赖:

```XML
<!--统一配置管理-->
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
  </dependency>
  <!--读取bootstrap文件-->
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-bootstrap</artifactId>
  </dependency>
  <!--seata-->
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
  </dependency>
```

**nacos配置**

```YAML
seata:
  registry: # TC服务注册中心的配置，微服务根据这些信息去注册中心获取tc服务地址
    type: nacos # 注册中心类型 nacos
    nacos:
      server-addr: 192.168.1.101:8848 # nacos地址
      namespace: "" # namespace，默认为空
      group: DEFAULT_GROUP # 分组，默认是DEFAULT_GROUP
      application: seata-server # seata服务名称
      username: nacos
      password: nacos
  tx-service-group: hmall # 事务组名称
  service:
    vgroup-mapping: # 事务组与tc集群的映射关系
      hmall: "default"
```

**添加共享配置（最后一行）**

```YAML
spring:
  application:
    name: trade-service # 服务名称
  profiles:
    active: dev
  cloud:
    nacos:
      server-addr: 192.168.150.101 # nacos地址
      config:
        file-extension: yaml # 文件后缀名
        shared-configs: # 共享配置
          - dataId: shared-seata.yaml # 共享seata配置
```

**启用Feign的支持**

```YAML
feign:
  sentinel:
    enabled: true # 开启Feign对Sentinel的整合
```

## XA模式

`XA` 规范 是` X/Open` 组织定义的分布式事务处理（DTP，Distributed Transaction Processing）标准，XA 规范 描述了全局的`TM`与局部的`RM`之间的接口，几乎所有主流的数据库都对 XA 规范 提供了支持。

**步骤**

一阶段：

- 事务协调者通知每个事务参与者执行本地事务
- 本地事务执行完成后报告事务执行状态给事务协调者，此时事务不提交，继续持有数据库锁

二阶段：

- 事务协调者基于一阶段的报告来判断下一步操作
- 如果一阶段都成功，则通知所有事务参与者，提交事务
- 如果一阶段任意一个参与者失败，则通知所有事务参与者回滚事务

**优缺点**

`XA`模式的优点是什么？

- 事务的强一致性，满足ACID原则
- 常用数据库都支持，实现简单，并且没有代码侵入

`XA`模式的缺点是什么？

- 因为一阶段需要锁定数据库资源，等待二阶段结束才释放，性能较差
- 依赖关系型数据库实现事务

### 实现XA模式

**启用XA**

```yaml
seata:
  data-source-proxy-mode: XA
```

`@GlobalTransactional`标记分布式事务的入口方法

子事务加上`@Transactional`

## AT模式

`AT`模式同样是分阶段提交的事务模型，不过缺弥补了`XA`模型中资源锁定周期过长的缺陷。

**步骤**

阶段一`RM`的工作：

- 注册分支事务
- 记录undo-log（数据快照）
- 执行业务sql并提交
- 报告事务状态

阶段二提交时`RM`的工作：

- 删除undo-log即可

阶段二回滚时`RM`的工作：

- 根据undo-log恢复数据到更新前

**AT与XA的区别**

简述`AT`模式与`XA`模式最大的区别是什么？

- `XA`模式一阶段不提交事务，锁定资源；`AT`模式一阶段直接提交，不锁定资源。
- `XA`模式依赖数据库机制实现回滚；`AT`模式利用数据快照实现数据回滚。
- `XA`模式强一致；`AT`模式最终一致

可见，AT模式使用起来更加简单，无业务侵入，性能更好。因此企业90%的分布式事务都可以用AT模式来解决。

### 实现AT

**启用XA**

```yaml
seata:
  data-source-proxy-mode: AT # 可以不填默认为AT模式
```

每一个对应的微服务库都需要有一个自己的`undo_log`

```sql
-- for AT mode you must to init this sql for you business database. the seata server not need it.
CREATE TABLE IF NOT EXISTS `undo_log`
(
    `branch_id`     BIGINT       NOT NULL COMMENT 'branch transaction id',
    `xid`           VARCHAR(128) NOT NULL COMMENT 'global transaction id',
    `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
    `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
    `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
    `log_created`   DATETIME(6)  NOT NULL COMMENT 'create datetime',
    `log_modified`  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8mb4 COMMENT ='AT transaction mode undo table';

```

# MQ(MessageQueue)

几种常见MQ的对比：

|            | RabbitMQ                | ActiveMQ                       | RocketMQ   | Kafka      |
| ---------- | ----------------------- | ------------------------------ | ---------- | ---------- |
| 公司/社区  | Rabbit                  | Apache                         | 阿里       | Apache     |
| 开发语言   | Erlang                  | Java                           | Java       | Scala&Java |
| 协议支持   | AMQP，XMPP，SMTP，STOMP | OpenWire,STOMP，REST,XMPP,AMQP | 自定义协议 | 自定义协议 |
| 可用性     | 高                      | 一般                           | 高         | 高         |
| 单机吞吐量 | 一般                    | 差                             | 高         | 非常高     |
| 消息延迟   | 微秒级                  | 毫秒级                         | 毫秒级     | 毫秒以内   |
| 消息可靠性 | 高                      | 一般                           | 高         | 一般       |

追求可用性：Kafka、 RocketMQ 、RabbitMQ

追求可靠性：RabbitMQ、RocketMQ

追求吞吐能力：RocketMQ、Kafka

追求消息低延迟：RabbitMQ、Kafka

## RabbitMQ

RabbitMQ是基于Erlang语言开发的开源消息通信中间件，官网地址：

https://www.rabbitmq.com/

### Docker安装

```shell
# 安装RabbitMQ镜像
docker load -i "E:/wfw/mq.tar"
# 安装容器
docker run -e RABBITMQ_DEFAULT_USER=hyh -e RABBITMQ_DEFAULT_PASS=123456 --name mq --hostname mq -p 15672:15672 -p 5672:5672 --network hm-net -d rabbitmq:3.8-management
# 配置挂载目录（windows下的Docker直接配置mq挂载目录会报错）
docker cp "E:/Docker/mq-plugins" mq:/plugins
```

**控制台地址：**http://192.168.1.104:15672

 其中包含几个概念：

- **`publisher`**：生产者，也就是发送消息的一方
- **`consumer`**：消费者，也就是消费消息的一方
- **`queue`**：队列，存储消息。生产者投递的消息会暂存在消息队列中，等待消费者处理
- **`exchange`**：交换机，负责消息路由。生产者发送的消息由交换机决定投递到哪个队列，只负责路由消息，无法存储消息。
- **`virtual host`**：虚拟主机，起到数据隔离的作用。每个虚拟主机相互独立，有各自的exchange、queue



**`exchanges`**

- **Bindings：**队列绑定关系规则
- **Publish message：**模拟发送消息



## Spring AMQP

官方地址：https://spring.io/projects/spring-amqp

### 入门案例

#### 引入依赖

```xml
<!--AMQP依赖，包含RabbitMQ-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

#### yaml配置

```yaml
spring:
  rabbitmq:
    host: 192.168.83.103 # 你的虚拟机IP
    port: 15672 # 端口
    virtual-host: /hmall # 虚拟主机
    username: hmall # 用户名
    password: 123 # 密码
```

#### 发送消息

```java
@SpringBootTest
class SpringAmqpTest {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    public void testSimplQueue(){
        // 1.队列名
        String queueName = "simple.queue";
        // 2.消息
        String message = "hello, spring amqp!";
        // 3.发送消息
        rabbitTemplate.convertAndSend(queueName, message);
    }
}
```

#### 接收消息

```java
@Slf4j
@Component
public class SpringRabbitListener {
    @RabbitListener(queues = "simple.queue")
    public void listenSimpleQueue(String message){
        log.info("接收到simple.queue的消息：【{}】",message);
    }

}

```

## Work Queues

Work queues，任务模型。简单来说就是**让多个消费者绑定到一个队列，共同消费队列中的消息**。

### 发送消息

```java
@Test
public void testWorkQueue(){
    // 1.队列名
    String queueName = "work.queue";

    for (int i = 0; i < 50; i++) {
        // 2.消息
        String message = "hello, spring amqp_" + i;
        // 3.发送消息
        rabbitTemplate.convertAndSend(queueName, message);
    }
}
```

### 接受消息

```java
@RabbitListener(queues = "work.queue")
public void listenWorkQueue1(String message){
    System.out.println("消费者1接收到work.queue的消息：" + message + "，" + LocalTime.now());
}
@RabbitListener(queues = "work.queue")
public void listenWorkQueue2(String message){
    System.err.println("消费者2 ....... 接收到work.queue的消息：" + message + "，" + LocalTime.now());
}
```

### 消息延迟

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 1 # 每次只消费一个消息
```

Work模型的使用：

- 多个消费者绑定到一个队列，同一条消息只会被一个消费者处理
- 通过设置prefetch来控制消费者预取的消息数量

## 交换机

交换机的类型有四种：

- **[Fanout](#Fanout交换机)**：广播，将消息交给所有绑定到交换机的队列。我们最早在控制台使用的正是Fanout交换机
- **[Direct](#Direct交换机)**：订阅，基于RoutingKey（路由key）发送给订阅了消息的队列
- **[Topic](#Topic交换机)**：通配符订阅，与Direct类似，只不过RoutingKey可以使用通配符
- **[Headers](#Headers交换机)**：头匹配，基于MQ的消息头匹配，用的较少。

### Fanout交换机

在广播模式下，消息发送流程是这样的：

- 1）  可以有多个队列
- 2）  每个队列都要绑定到Exchange（交换机）
- 3）  生产者发送的消息，只能发送到交换机
- 4）  交换机把消息发送给绑定过的所有队列
- 5）  订阅队列的消费者都能拿到消息

**发送消息**

```java
@Test
public void testFanoutQueue(){
    // 1.队列名
    String exchangeName = "hmall.fanout";
    // 2.消息
    String message = "hello, everyone!";
    // 3.发送消息 三个参数
    rabbitTemplate.convertAndSend(exchangeName, "", message);
}
```

**接受消息**

```java
@RabbitListener(queues = "fanout.queue1")
public void listenFanoutQueue1(String message){
    log.info("消费者1接收到fanout.queue的消息：【{}】",message);
}

@RabbitListener(queues = "fanout.queue2")
public void listenFanoutQueue2(String message){
	log.info("消费者2接收到fanout.queue的消息：【{}】",message);
}
```

### Direct交换机

在Direct模型下：

- 队列与交换机的绑定，不能是任意绑定了，而是要指定一个`RoutingKey`（路由key）
- 消息的发送方在 向 Exchange发送消息时，也必须指定消息的 `RoutingKey`。
- Exchange不再把消息交给每一个绑定的队列，而是根据消息的`Routing Key`进行判断，只有队列的`Routingkey`与消息的 `Routing key`完全一致，才会接收到消息

**发送消息**

```javascript
@Test
public void testDirectQueue(){
    // 1.队列名
    String exchangeName = "hmall.direct";
    // 2.消息
    String routingKey = "blue";
    String message = "hello, " + routingKey;
    // 3.发送消息
    rabbitTemplate.convertAndSend(exchangeName, routingKey, message);
}
```

**接受消息**

```java
@RabbitListener(queues = "direct.queue1")
public void listenDirectQueue1(String message){
    log.info("消费者1接收到direct.queue的消息：【{}】",message);
}

@RabbitListener(queues = "direct.queue2")
public void listenDirectQueue2(String message){
    log.info("消费者2接收到direct.queue的消息：【{}】",message);
}
```

### Topic交换机

`Topic`类型的`Exchange`与`Direct`相比，都是可以根据`RoutingKey`把消息路由到不同的队列。

只不过`Topic`类型`Exchange`可以让队列在绑定`BindingKey` 的时候使用通配符！

```
BindingKey` 一般都是有一个或多个单词组成，多个单词之间以`.`分割，例如： `item.insert
```

通配符规则：

- `#`：匹配一个或多个词
- `*`：匹配不多不少恰好1个词

举例：

- `china.#`：能够匹配`china.ZheJiang.weather` 或者 `china.weather`
- `china.*`：只能匹配`china.weather`或者 `china.news`
- `#.weather`：能够匹配`china.ZheJiang.weather` 或者 `china.weather`
- `*.weather`：能够匹配`china.weather` 或者 `japan.weather`

**发送消息**

```java
@Test
public void testTopicQueue(){
    // 1.队列名
    String exchangeName = "hmall.topic";
    // 2.消息
    String routingKey = "china.weather";
    String message = "hello, " + routingKey;
    // 3.发送消息
    rabbitTemplate.convertAndSend(exchangeName, routingKey, message);
}
```

**接收消息**

```java
@RabbitListener(queues = "topic.queue1")
public void listenTopicQueue1(String message){
    log.info("消费者1接收到topic.queue的消息：【{}】",message);
}

@RabbitListener(queues = "topic.queue2")
public void listenTopicQueue2(String message){
    log.info("消费者2接收到topic.queue的消息：【{}】",message);
}
```

### 声明队列和交换机

SpringAMQP提供了几个类，用来声明队列，交换机及绑定关系：

- **Queue：**用于声明队列，可以用工厂类`QueueBuilder`构建
- **Exchange：**用于声明交换机，可以用工厂类`ExchangeBuilder`构建
- **Binding：**用于声明队列和交换机的绑定关系，可以用工厂类`BindingBuilder`构建

**基于Bean声明队列和交换机**

```java
package com.itheima.consumer.config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FanoutConfig {
     /**
     * 声明交换机
     * @return Fanout类型交换机
     */
    @Bean
    public FanoutExchange fanoutExchange(){
        return new FanoutExchange("hmall.fanout");
//        return ExchangeBuilder.fanoutExchange("hmall.fanout").build();
    }
    /**
     * 第1个队列
     */
    @Bean
    public Queue fanoutQueue1(){
        return new Queue("fanout.queue1");
//        return QueueBuilder.durable("fanout.queue1").build();
    }
     /**
     * 绑定队列和交换机
     */
    @Bean
    public Binding fanoutQueue1Binding(Queue fanoutQueue1,FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue1).to(fanoutExchange);
    }
     /**
     * 第2个队列
     */
    @Bean
    public Queue fanoutQueue2(){
        return new Queue("fanout.queue2");
//        return QueueBuilder.durable("fanout.queue1").build();
    }
     /**
     * 绑定队列和交换机
     */
    @Bean
    public Binding fanoutQueue2Binding(Queue fanoutQueue2,FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue2).to(fanoutExchange);
    }
}
```

**基于注解声明队列和交换机**

SpringAMQP还提供了基于@RabbitListener注解来声明队列和交换机的方式：

```
@RabbitListener(bindings = @QueueBinding(
	value = @Queue(name = "direct.queue1", durable = "true"),
	exchange = @Exchange(name = "hmall.direct", type = ExchangeTypes.DIRECT),
	key = {"red", "blue"}
))
public void listenDirectQueue1(String message){
	log.info("消费者1接收到direct.queue的消息：【{}】",message);
}

@RabbitListener(bindings = @QueueBinding(
	value = @Queue(name = "direct.queue2", durable = "true"),
	exchange = @Exchange(name = "hmall.direct", type = ExchangeTypes.DIRECT),
	key = {"red", "yellow"}
))
public void listenDirectQueue2(String message){
	log.info("消费者2接收到direct.queue的消息：【{}】",message);
}
```

## 消息转换器

Spring的消息发送代码接收的消息体是一个Object：

而在数据传输时，它会把你发送的消息序列化为字节发送给MQ，接收消息的时候，还会把字节反序列化为Java对象。

只不过，默认情况下Spring采用的序列化方式是JDK序列化。众所周知，JDK序列化存在下列问题：

- 数据体积过大
- 有安全漏洞
- 可读性差























在线文档：https://b11et3un53m.feishu.cn/wiki/space/7229522334074372099?ccm_open_type=lark_wiki_spaceLink&open_tab_from=wiki_home











