---
title: "Macula Boot Starter Cloud Gateway"
linkTitle: "Gateway"
weight: 3
---
## 概述

网关服务模块，给每个平台应用依赖用的。主要提供token认证、鉴权、接口加解密等功能。



## 组件坐标

```xml

<dependency>
    <groupId>dev.macula.boot</groupId>
    <artifactId>macula-boot-starter-cloud-alibaba</artifactId>
    <version>${macula.version}</version>
</dependency>

<dependency>
<groupId>dev.macula.boot</groupId>
<artifactId>macula-boot-starter-cloud-alibaba-scg</artifactId>
<version>${macula.version}</version>
</dependency>
```



## 使用配置

```yaml
spring:
  gateway:
    routes:
      - id: macula-cloud-system
        uri: lb://macula-cloud-system
        predicates:
          - Path=/system/**
        filters:
          - StripPrefix=1
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          client-id: e4da4a32-592b-46f0-ae1d-784310e88423
          client-secret: secret
          introspection-uri: http://127.0.0.1:9010/oauth2/introspect     
  redis:																# 网关自己的redis配置
    database: 0										
    host: 127.0.0.1
    port: 6379
    system:															# macula-cloud的system模块的redis配置
      database: 0
      host: 127.0.0.1
      port: 6379
macula:
  gateway:
    sign-switch: true                       # 接口签名全局开关，默认true
    force-sign: false                       # 是否强制校验指定URL的接口签名，默认false
    crypto-switch: true           			# 接口加解密全局开关，默认true
    force-crypto: false           			# 是否强制校验指定URL的接口要不要加解密，默认false
    protect-urls: 						    # 需要保护的URL，前端可通过/gateway/protect/urls获取
      crypto:                               # 加密
        - /system/xxx/**
        - /mall/api/v1/xxx/**
      sign:                                 # 签名
        - /system/xxx/*
        - /mall/api/v1/**
    security:
      ignore-urls: /usr/xxx,/bbb/xxx  		# 忽略认证的路径，Ant Path格式
      only-auth-urls: /usr/xx, /bbb/xxx 	# 仅需认证无需鉴权的路径
```



## 核心功能

### Token认证

将oauth2的token转换为JWT传递给微服务，微服务通过JWT获取用户信息和角色信息。

1. 前端访问gateway接口，在HTTP请求头添加` Authorization Bear xxxxx`
2. gateway收到token后，调用iam服务器的introspect url返回用户信息和角色信息（同时会以token有效期来缓存用户信息）
3. 通过AddJwtFilter将用户信息和角色信息转为JWT Token放入请求头给后续的微服务的安全模块校验认证（JWT也会缓存）

### AK/SK认证

将hmac的签名等信息转为JWT传递给微服务，微服务通过JWT获取用户信息和角色信息。

1. 如果请求携带hmac信息，则校验签名，根据appId检查应用，如果应用有租户ID，则设置到租户上下文
2. 通过AddJwtFilter生成JWT，具体同上

### URL安全

根据用户的角色和URL所需角色对比，控制URL权限。

1. macula-cloud-system模块在维护菜单、角色等信息后会定时将URL和角色关系缓存到redis

2. gateway根据请求URL匹配redis中的URL角色关系找出访问该URL所需角色

3. 根据当前用户的角色列表是否满足上一步的角色要求决定是否放行

{{% alert title="提示" color="primary" %}}

可以配置URL是否认证或者鉴权

{{% /alert %}}



###  定制JWT的Claims

框架开放了JwtClaimsCustomizer接口，可以用来添加你自己的Claim

```java
@Bean
JwtClaimsCustomizer jwtClaimsCustomizer() {
    return builder -> {
        builder.claim("sal", "demo");
        builder.claim("abc", "aaa");
    };
}
```



### 接口的加解密和签名

网关要支持接口加解密和签名的话，首先要实现CryptoService，加解密所需方法。比如接入密钥服务系统。

```java
/**
 * {@code CryptoService} 接口加解密服务
 *
 * @author rain
 * @since 2023/3/22 19:36
 */
public interface CryptoService {

    /**
     * 获取用于加密前端生成的SM4Key的公钥
     *
     * @return 公钥
     */
    String getSm2PublicKey();

    /**
     * 解密前端传过来经过非对称加密的SM4 KEY
     *
     * @param key 加密过的sm4 key
     * @return SM4KEY明文
     */
    String decryptSm4Key(String key);

    /**
     * 加密数据
     *
     * @param plainText 明文
     * @param sm4Key    加密的密钥
     * @return base64密文
     */
    String encrypt(String plainText, String sm4Key);

    /**
     * 解密数据
     *
     * @param secretText base64密文
     * @param sm4Key     解密的密钥
     * @return 明文
     */
    String decrypt(String secretText, String sm4Key);

}
```

本地加解密实现示例如下：

```java
/**
 * {@code CryptoLocaleServiceImpl} 本地加解密服务
 *
 * @author rain
 * @since 2023/3/23 22:01
 */
@Component
@Slf4j
public class CryptoLocaleServiceImpl implements CryptoService, InitializingBean {
    private SM2 sm2;

    @Override
    public String getSm2PublicKey() {
        return HexUtil.encodeHexStr(sm2.getPublicKey().getEncoded());
    }

    @Override
    public String decryptSm4Key(String key) {
        return sm2.decryptStr(key, KeyType.PrivateKey);
    }

    @Override
    public String encrypt(String plainText, String sm4Key) {
        return SmUtil.sm4(sm4Key.getBytes(StandardCharsets.UTF_8)).encryptBase64(plainText);
    }

    @Override
    public String decrypt(String secretText, String sm4Key) {
        return SmUtil.sm4(sm4Key.getBytes(StandardCharsets.UTF_8)).decryptStr(secretText);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        KeyPair pair = SecureUtil.generateKeyPair("SM2");
        sm2 = SmUtil.sm2(pair.getPrivate(), pair.getPublic());
        if (log.isDebugEnabled()) {
            log.debug("sm2 public key: {}", HexUtil.encodeHexStr(sm2.getPublicKey().getEncoded()));
            log.debug("sm2 private key: {}", HexUtil.encodeHexStr(sm2.getPrivateKey().getEncoded()));
            log.debug("sm4 encrypted key: {}", sm2.encryptBase64("1234567890abcdef", KeyType.PublicKey));
        }
    }
}
```

#### 前端流程

- 密钥协商
    - 前端获取需要加密或者签名的接口：/gateway/protect/urls，返回{ crypto: [], sign: [] }，如果与当前请求匹配，则执行下述流程
    - 前端获取公钥：/gateway/protect/key
    - 前端根据URL规则判断是否要加密和签名
    - 前端随机产生一串密钥key并使用SM4公钥加密，将该密钥放入HTTP请求头`sm4-key`
- 请求加密
    - 前端对GET的Query参数param1=value1&param2=value2用上述随机串key进行SM4加密（注意param要排序）
    - 前端对POST的JSON Body用上述key进行SM4加密
    - 前端GET请求的加密参数附加在URL?data=xxx中，POST请求的加密参数也是以JSON格式放在data这个key中
    - 加密后以Base64编码（GET请求要encodeURI，Base64含有+=/等符号）
    - 请求头添加sym-alg，标识加密算法SM4
- 请求签名（加密后的，非空参数值才参与签名）
    - 生成当前时间戳，为UTC 1970年1月1日0时开始的毫秒数(Unix 时间戳)
    - 随机生成nonce随机串，注意要保证随机唯一性
    - GET请求签名sha256(path+param1=value1&param2=value2...+key+timestamp+nonce)
    - POST请求签名sha256(POST签名体 = path+param1=value1&param2=value2...+SHA-256=sha256(body))
      +key+timestamp+nonce)
    - SHA256是16进制字符串格式
    - timestamp、signature、nonce、algorithm（默认SHA-256）放入Header
    - 签名算法支持MD2、MD5、SHA-1、SHA-256
- 响应解密
    - 响应体的加密内容在JSON串的data这个key中，使用SM4解密

#### 后端流程

- 验证签名
    - 后端根据URL规则和macula.gateway.force-sign判断是否强制验证签名
    - 后端根据验签规则验证签名，不通过则返回错误
- 请求解密
    - 后端根据macula.gateway.force-crypto判断是否强制加解密，如果请求URL在列表中但是没有携带sm4-key则返回错误
    - 后端解密请求数据，然后将加密的返回数据替换Result的data
- 响应加密
    - 根据需要对响应加密，放入Result的data字段中

### 配置System的Redis

网关的URL与角色对应关系数据、应用数据是缓存在macula-cloud的system模块的redis中，需要配置system的redis。利用多redis配置方式来进行配置：

```java
/**
 * {@code RedisConfiguration} Redis配置
 *
 * @author rain
 * @since 2023/4/21 11:50
 */
@Configuration
public class RedisConfiguration {
    @Bean
    @ConfigurationProperties(prefix = "spring.redis")
    public RedisProperties redisProperties() {
        return new RedisProperties();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.redis.system")
    public RedisProperties sysRedisProperties() {
        return new RedisProperties();
    }

    @Primary
    @Bean(destroyMethod = "shutdown")
    public RedissonClient redissonClient(ApplicationContext ctx, RedisProperties redisProperties) throws Exception {
        Config config = RedissonConfigBuilder.create().build(ctx, redisProperties, new RedissonProperties());
        return Redisson.create(config);
    }

    @Bean(destroyMethod = "shutdown")
    public RedissonClient sysRedissonClient(ApplicationContext ctx, RedisProperties sysRedisProperties)
        throws Exception {
        Config config = RedissonConfigBuilder.create().build(ctx, sysRedisProperties, new RedissonProperties());
        return Redisson.create(config);
    }

    @Bean(name = "sysRedisTemplate")
    public RedisTemplate<String, Object> sysRedisTemplate(
        @Qualifier("sysRedissonClient") RedissonClient sysRedissonClient) {
        //数据泛型类型
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        //设置连接工厂（Jedis或Lettuce）
        template.setConnectionFactory(new RedissonConnectionFactory(sysRedissonClient));

        //设置key的序列化方式---String
        template.setKeySerializer(RedisSerializer.string());
        template.setHashKeySerializer(RedisSerializer.string());
        //初始化RedisTemplate的参数设置
        template.afterPropertiesSet();
        return template;
    }
}
```

### 开启CORS

```yaml
spring:
  cloud:
    gateway:
      globalcors:  												# 全局CORS配置，适用于所有路由
        cors-configurations:
          '[/**]':  											# 匹配所有路径的请求，这里的'/**'表示所有路径
            allowed-origin-patterns: "*"  # 允许所有来源（使用通配符*），可根据需要限制特定域名
            allowed-methods: "*"  				# 允许所有HTTP方法（GET, POST, PUT, DELETE等），可以指定具体的方法
            allowed-headers: "*"  				# 允许所有请求头，或指定特定请求头
            allow-credentials: true  			# 是否允许客户端发送cookie或其他凭证
            max-age: 1800  								# 预检请求的缓存时间，单位为秒（1800秒=30分钟），在这个时间内，浏览器可以缓存CORS的预检请求结果，不必每次都发送预检请求
        add-to-simple-url-handler-mapping: true  # 是否将CORS配置应用于简单的URL处理器映射（通常用于非路由路径）
```



## 依赖引入

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>

    <dependency>
        <groupId>dev.macula.boot</groupId>
        <artifactId>macula-boot-starter-redis</artifactId>
    </dependency>

    <dependency>
        <groupId>dev.macula.boot</groupId>
        <artifactId>macula-boot-commons</artifactId>
    </dependency>

    <dependency>
        <groupId>com.nimbusds</groupId>
        <artifactId>oauth2-oidc-sdk</artifactId>
    </dependency>

    <dependency>
        <groupId>com.github.ben-manes.caffeine</groupId>
        <artifactId>caffeine</artifactId>
    </dependency>
</dependencies>
```

## 版权说明

- oauth2-oidc-sdk：https://github.com/hidglobal/oauth-2.0-sdk-with-openid-connect-extensions/blob/master/LICENSE.txt
- caffeine：https://github.com/ben-manes/caffeine/blob/master/LICENSE
- spring-cloud-gateway：https://github.com/spring-cloud/spring-cloud-gateway/blob/main/LICENSE.txt