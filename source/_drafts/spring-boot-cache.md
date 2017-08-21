---
title: 'spring-boot中方便的应用cache缓存'
tags: spring-boot
categories: spring-boot
---
>`Cache`(缓存), 提供快速的方式多次读取相同的数据。`cache`与`buffer`的区别，`buffer`(缓冲)，作为快速和慢速对象之间的数据的中间临时存储。

`Spring Framework`从3.1版本开始，支持通明缓存;到了4.1版本，明确支持`JSR-107`注解标准。`Spring Framework`中`cache`抽象是通过`org.springframework.cache.Cache`和`org.springframework.cache.CacheManager`接口实现的。
而在`Spring Boot`中，更添加了对`cache`的[快速配置][p],接下来如何在`Spring Boot`中开启`cache`?

[p]:http://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html

## 引入缓存

- 在`pom.xml`文件中添加相关`starters`依赖；如果想应用`JSR-107`注解标准,需要引入`javax.cache:cache-api`

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        <!--JSR-107-->
        <dependency>
            <groupId>javax.cache</groupId>
            <artifactId>cache-api</artifactId>
        </dependency>

- 在`Spring Boot`主类中添加注解声明`@EnableCaching`开启缓存;如果应用缓存的类不是基于接口方式，开启`proxyTargetClass`属性

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
        //另，如使用@CacheResult(cacheName = "names")注解，需在方法实现上
        @Override
        public int calcPrice(int num) {
            logger.debug("计算数值 {} + 4 = {}", num, num + 4);
            return num+4;
        }
- 进行单元测试，会看到其中第二次调用方法，日志没有打印，计算结果从缓存中获得。
        @RunWith(SpringJUnit4ClassRunner.class)
        @SpringBootTest
        public class AppTest {
            @Autowired
            private CalcPrice calcPrice;
            @Test
            public void testCache(){
                assert calcPrice.calcPrice(5) == 9;
        //        assert calcPrice.delPrice(5) == true;
                //第二次查询
                assert calcPrice.calcPrice(5) == 9;
            }
        }
        日志结果:（第二次没有打印，没有进入方法体内）
        2017-08-15 10:26:04,071 [main] INFO  org.illuwater.AppTest - Started AppTest in 6.858 seconds (JVM running for 11.217)
        2017-08-15 10:26:05,027 [main] INFO  org.illuwater.service.CalcPriceImpl - 计算数值 5 + 4 = 9
        2017-08-15 10:26:05,083 [Thread-2] INFO  org.springframework.web.context.support.GenericWebApplicationContext - Closing org.springframework.web.context.support.GenericWebApplicationContext@6e0f5f7f: startup date [Tue Aug 15 10:25:58 CST 2017]; root of context hierarchy

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

相应的`JSR-107`注解的比较，请看[官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html#cache-jsr-107)

## 支持的缓存提供程序

`cache`抽象框架并不会真正提供缓存的仓库，并且如果你并没有明确配置`CacheManager`,那`Spring Boot`会自动安装以下顺序侦查：
- `Generic`
- `JCache(JSR-107)(Ehcache 3, Hazelcast, Infinispan 等)`
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

## 支持`EhCache 2.x`, 步骤

- 添加`EhCache`依赖
- 添加`ehcache.xml`配置文件，定义相关的缓存策略等
- `spring.cache.ehcache.config=classpath:ehcache.xml`
