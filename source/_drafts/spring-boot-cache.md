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