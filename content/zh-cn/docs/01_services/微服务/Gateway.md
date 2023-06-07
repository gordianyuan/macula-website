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
        jwt:
          issuer-uri: http://127.0.0.1:9010
        opaquetoken:
          client-id: e4da4a32-592b-46f0-ae1d-784310e88423
          client-secret: secret
          introspection-uri: http://127.0.0.1:9010/oauth2/introspect          
macula:
  gateway:
    crypto-switch: true           			# 接口加解密全局开关，默认true
    force-crypto: false           			# 是否强制校验接口要不要加解密，默认false
    crypto-urls: 							# 需要加解密的URL，前端可通过/gateway/crypto/urls获取
      - /system/xxx/**
      - /mall/api/v1/xxx/**
    security:
      ignore-urls: /usr/xxx,/bbb/xxx  		# 忽略认证的路径，Ant Path格式
      only-auth-urls: /usr/xx, /bbb/xxx 	# 仅需认证无需鉴权的路径
      jwt-secret: xxx  						# JWT的签名密钥
spring:
  redis:
    xxx         							# 通过redis获取URL-ROLES对应数据
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

### 接口的加解密

网关要支持接口加解密的话，首先要实现CryptoService，加解密所需方法。比如接入密钥服务系统。

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

接口加解密流程：

- 前端获取需要加密的接口：/gateway/crypto/urls
- 前端获取公钥：/gateway/crypto/key
- 前端随机产生一串密钥并使用公钥加密，将改密钥放入HTTP请求头`sm4-key`
- 后端根据macula.gateway.force-crypto判断是否强制加解密，如果请求URL在列表中但是没有携带sm4-key则返回错误
- 前端GET请求的加密参数附加在URL?data=xxx中，POST请求的加密参数也是以JSON格式放在data这个key中
- 后端解密请求数据，然后将加密的返回数据替换Result的data



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
