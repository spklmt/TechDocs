# Nacos

>Nacos (Dynamic Naming and Configuration Service)，提供动态服务发现、配置管理和服务管理平台。

## 1. 注册中心

### 1.1 服务注册

#### 1.1.1 配置注册中心

```yaml
spring:
  application:
    name: ${server-name}
  cloud:
    nacos:
      discovery:
        server-addr: ${ip:port}
```

#### 1.1.2 开启服务注册/发现功能

```java
@EnableDiscoveryClient
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

[注] 导入 `spring-cloud-starter-alibaba-nacos-discovery` 依赖

### 1.2 服务发现

#### 1.2.1 DiscoveryClient

```java
@Autowired
DiscoveryClient discoveryClient;

public interface DiscoveryClient extends Ordered {
	// ...
    
    List<ServiceInstance> getInstances(String serviceId);

    List<String> getServices();
    
    // ...
}
```

#### 1.2.2 NacosServiceDiscovery

```java
@Autowired
NacosServiceDiscovery nacosServiceDiscovery;

public class NacosServiceDiscovery {
	// ...
    
    public List<ServiceInstance> getInstances(String serviceId) throws NacosException {
        String group = this.discoveryProperties.getGroup();
        List<Instance> instances = this.namingService().selectInstances(serviceId, group, true);
        return hostToServiceInstanceList(instances, serviceId);
    }

    public List<String> getServices() throws NacosException {
        String group = this.discoveryProperties.getGroup();
        ListView<String> services = this.namingService().getServicesOfServer(1, Integer.MAX_VALUE, group);
        return services.getData();
    }
    
    // ...
}
```

### 1.3 负载均衡

#### 1.3.1 LoadBalancerClient

```java
@Autowired
LoadBalancerClient loadBalancerClient;

// 负载均衡选择服务实例
loadBalancerClient.choose("${server-name}");
```

#### 1.3.2 注解负载均衡

```java
@Configuration
public class ServiceConfig {

    // String url = "http://${server-name}/api/" 负载均衡选择服务实例
    @LoadBalanced
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```

[注] 导入 `spring-cloud-starter-loadbalancer` 依赖

## 2. 配置中心

### 2.1 配置

```yaml
spring:
  application:
    name: service-product
  profiles:
    active: dev
  cloud:
    nacos:
      server-addr: 127.0.0.1:8848
      config:
        namespace: ${spring.profiles.active:public}
        import-check:
          enabled: false

---
spring:
  config:
    import:
      - nacos:config1.yaml?group=group1
      - nacos:config2.yaml?group=group1
    activate:
      on-profile: dev
---
spring:
  config:
    import:
      - nacos:config1.yaml?group=group1
      - nacos:config2.yaml?group=group1
      - nacos:config3.yaml?group=group1
    activate:
      on-profile: prod
```

[注] 导入 `spring-cloud-starter-alibaba-nacos-config` 依赖

### 2.2 动态刷新

> 加载动态配置

#### 2.2.1 @RefreshScope

```java
@RefreshScope
@RestController
public class DemoController {

    @Value("${test.timeout}")
    String timeout;
}
```

#### 2.2.2 ConfigurationProperties

```java
@Component
@ConfigurationProperties(prefix = "test") // 配置批量绑定, ⽆需 @RefreshScope 实现⾃动刷新
@Data
public class DemoProperties {
    String timeout;
}

@RestController
public class DemoController {

    @Autowired
    DemoProperties demoProperties;
}
```

### 2.3 NacosConfigManager

```java
@EnableDiscoveryClient
@SpringBootApplication
public class OrderMainApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderMainApplication.class, args);
    }

    // 监听配置文件
    @Bean
    ApplicationRunner applicationRunner(NacosConfigManager nacosConfigManager) {
        return args -> {
            ConfigService configService = nacosConfigManager.getConfigService();
            configService.addListener("service-test.yaml", "DEFAULT_GROUP", new Listener() {

                @Override
                public Executor getExecutor() {
                    return Executors.newFixedThreadPool(4);
                }

                @Override
                public void receiveConfigInfo(String configInfo) {
                    System.out.println("configInfo:" + configInfo);
                }
            });
        };
    }
}
```

### 2.4 namespace, dataId, group

#### 2.4.1 namespace

命名空间：实现多环境隔离（local, development, test, beta, production）

#### 2.4.2 dataId

数据集id：配置文件（name.suffix）

#### 2.4.3 groupId

分组id：一般以微服务名字作为分组。

## 面试题

### 1. 如果注册中心宕机，远程调用是否可以成功？

- 服务从未调用过注册中心，立即失败：`com.alibaba.nacos.api.exception.NacosException: Client not connected, current status:UNHEALTHY`

- 服务调用过注册中心，注册中心宕机，实例未宕机，服务根据缓存实例名单，调用成功
- 服务调用过注册中心，注册中心和实例都宕机，服务根据缓存实例名单，调用失败：`java.net.ConnectException: Connection refused: connect`



