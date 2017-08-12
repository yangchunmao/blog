---
title: 'spring-boot:cache'
tags: spring-boot
categories: spring-boot
---
>Cache, 缓存 解决因响应慢导致影响用户体验的问题。缓存用好很牛，用不好也会导致很多其他相关问题。
Spring boot 提供对于缓存的自动配置的支持，使得可以更加方便的应用缓存。


## 引入缓存

- 在`pom.xml`文件中添加相关`starters`依赖,`spring cache`是基于`JSR-107 API`标准的

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>

- 在`spring boot`主类中添加注解声明`@EnableCaching`开启缓存

        @EnableCaching
        @SpringBootApplication
        public class App {
            public static void main( String[] args ) {
                SpringApplication.run(App.class, args);
            }
        }

- 在数据访问接口中，增加缓存配置注解，如

        @CacheConfig(cacheNames = "prices")
        public interface CalcPrice {
            @Cacheable
            public int calcPrice(int num);
        }

        //实现方法为简单的logger日志输出
        @Override
        public int calcPrice(int num) {
            logger.debug("计算数值 {} + 4 = {}", num, num + 4);
            return num+4;
        }
- 进行单元测试，会看到其中第二次调用方法，日志没有打印，计算结果从缓存中获得。





## Cache注解详解

- `@CacheConfig`: 配置在类上的方法公用的缓存配置。可以配置缓存对象的名称，如上例中`@CacheConfig(cacheNames="prices")`, 还可以配置缓存的`cacheManager`。也可只用注解`Cacheable`。
- `@Cacheable`: 配置在类，函数上对其返回值进行缓存。以下是几个重要的属性。

属性 | 是否必须 | 描述
--- | :---: | :---
`value`, `cacheNames` | 非 | 连个等同的参数，指定缓存存储的集合名
`key` | 非 | 缓存对象存储在Map集合中的key值。缺省按函数所有参数组合，若自己配置需用[`SpEL`][]表达式
`condition` | 非 | 缓存对象条件，只有满足[`SpEL`][]表达式才会被缓存。
`unless` | 非 | 缓存对象条件，也需要[`SpEL`][]表达式，不同于`condition`参数地方在于它的判断时机，该条件在函数被调用后才判断，所有可以对`result`进行判断
`keyGenerator` | 非 | 用于指定`key`生成器，与参数`key`互斥，需要实现`org.springframework.cache.interceptor.KeyGenerator`接口
`cacheManager` | 非 | 用于指定那个缓存管理器，可以配置多个缓存管理器
`cacheResolver` | 非 | 用于指定使用那个缓存解析器，需要实现`org.springframework.cache.interceptor.CacheResolver`接口来实现自己的缓存解析器

[`SpEl`]:http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html#cache-spel-context

- `@CachePut`: 配置在函数、类上，根据参数定义进行缓存，与`@Cacheable`参数相同，不同的是，它每次都会真调用函数，所有主要用于数据新增和修改操作上。
- `@CacheEvict`: 配置在函数、类上，通过用在删除方法上，用来从缓存中异常数据，参数同`@Cacheable`,多了另2个参数
  + `allEntries`: 非必须，默认为false。当true时，会移除所有数据
  + `beforeInvocation`: 非必须，默认为false, 会在方法调用之后删除数据。反之为true。

## 支持的缓存插拔框架

- `JCacche`
- `EhCache` `EhCache`是一个纯`Java`的进程内缓存框架，具有快速、精干等特点，是Hibernate中默认的`CacheProvider`
- `Hazelcast` 一个高度可扩展的数据分发和集群平台，可用于实现分布式数据存储、数据缓存等
- `Infinispan` 是一个开源的数据网格平台，支持`JSR-107`标准的分布式缓存框架
- `Couchbase` Nosql数据库，前身是couchDB和memBase，对等网设计的，分布式缓存框架，整个集群没有单点失效，并且支持完全平行扩展。拥有专业的web管理界面，Restful API等
- `Redis` 
- `Caffeine` `java8`重写`Guava`的缓存，并将在`spring boot 2.0`中取代`Guava`的本地高效缓存
- `Guava(deprecated)`
- `Simple` 如果没有其他缓存`providers`, `spring boot`会通过`ConcurrentHashMap`作为缓存仓库
- `None` 当配置`@EnableCacheing`,缓存开启；如何不希望缓存应用当前的环境，可以强制配置缓存类型为`none`,如
        spring.cache.type=none


## 支持`EhCache 2.x`, 步骤：

- 添加`EhCache`依赖
- 添加`ehcache.xml`配置文件，定义相关的缓存策略等
- `spring.cache.ehcache.config=classpath:ehcache.xml`