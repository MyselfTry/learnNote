**Spring Security**

# 介绍

## 常用的功能

### 

![desc](media/ccc82d0c44eb5bbd40eb495f51e9bdb6.png)

## spring-security.xml

### 

![desc](media/2b8ae44546f247441fea40f4cb050247.png)

-   

    ![desc](media/0ea967f01432d26b9637eff348b7d07a.png)

### Spring 加载这个配置文件

-   

    ![desc](media/61c6df0a18fb5712d1233ccf5eea8788.png)

### 

![desc](media/f216f20a497ef12b46d9d3d4fecafe4d.png)

# 登录

## form-login 元素介绍http 元素下的 form-login 元素是用来定义表单登录信息的。当我们什么属性都不指定的时候 Spring Security 会为我们生成一个默认的登录页面。如果不想使用默认的登录页面，我们可以指定自己的登录页面。

## 使用自定义登录页面

### 

![desc](media/090f09001685a596f906b673296a8514.png)

### 

![desc](media/8f92618dc60c15265ab9a80d0e036cca.png)

## 指定登录后的页面

### default-target-url 指定

-   

    ![desc](media/0b3caa7c37b7859b8c2cb1a87f62eed2.png)

### authentication-success-handler-ref 指定

-   

    ![desc](media/7852a27847c0ed4dfa91b0103f0eca7c.png)

-   

    ![desc](media/fa653218f887a10551728e91fb6e57fe.png)

## 指定登录失败后的页面

### authentication-failure-url 指定

-   

    ![desc](media/ae5a7f962703bd475c3dd4174130238b.png)

### authentication-failure-handler-ref 指定

-   

    ![desc](media/50b0ecb3d2da07d2932c17e6542123e0.png)

## http-basic

### 

![desc](media/85a18e1c3eb4dda3be5f4ffb0b9139f8.png)

### 需要注意的是当我们同时定义了 http-basic 和 form-login 元素时，form-login 将具有更高的优先级。即在需要认证的时候 Spring Security 将引导我们到登录页面，而不是弹出一个窗口。

# 核心类简介

## Authentication

### 

![desc](media/0dfcef6f523bf2381f8470bf351877fc.png)

## SecurityContextHolder

### 

![desc](media/b72563598a62c8eb43959dd09b7f6bb7.png)

### 

![desc](media/5270d8c944c9dae262804b824124ef71.png)

## AuthenticationManager 和 AuthenticationProvider

### 

![desc](media/bc5fbeca3c59b05af5fe922aa3a80de9.png)

## UserDetailsService

### 

![desc](media/c73922c8339e4f2a8644615d57d792b7.png)

## JdbcDaoImpl

### 

![desc](media/69adffd0caed03df1dad19a54323bfb7.png)

### 

![desc](media/e914911192fa02db68738dca726ea669.png)

## InMemoryDaoImpl

### 

![desc](media/25baf7f870df8954e17083d87e678663.png)

## GrantedAuthority

### 

![desc](media/ef8216f9cfd5a926ea9eb25475b3c535.png)

# 认证简介

## 

![desc](media/86c18cde9e9ecf1991a4c83f5f3abb76.png)

## ExceptionTranslationFilter

![desc](media/beaeed2e17fc6a0e65f6ed73635a0ebf.png)

## 在 request 之间共享 SecurityContext

![desc](media/d84dab87b57ff379bf0bd557ed20d9c7.png)

# 异常信息本地化

## 

![desc](media/93a06709a72a4b4152da712c7d557edb.png)

# AuthenticationProvider

## 自定义

### 

![desc](media/6a67b013234fd96c9ea8721c8aa16ee4.png)

## 用户信息从数据库获取

### 使用 jdbc-user-service 获取

-   

    ![desc](media/5eb13e4e8470632c8967d83caa2922eb.png)

-   

    ![desc](media/8c48930d2d510890f1c6fdc531263f18.png)

-   

    ![desc](media/beaadb364a897099d5035848514797af.png)

### 直接使用 JdbcDaoImpl

-   

    ![desc](media/b0575ea87fd9eed7ff9adf734edd9523.png)

-   用户权限和用户组权限

    •

    ![desc](media/1408a8a15124b2a2fb3b481fcab82710.png)

## PasswordEncoder

### 使用内置的 PasswordEncoder

-   

    ![desc](media/a4d4f549e1d0f90414a355fb972a75db.png)

### salt-source 的示例

-   

    ![desc](media/7604c7d31e7ec7be5b9ce988f9ca5a02.png)

# 缓存 UserDetails

## CachingUserDetailsService

![desc](media/dcedbbe019a8f9bc74bc3de59f1afdaf.png)

## EhCacheBasedUserCacheEhCacheBasedUserCache 所引用的 Ehcache 是空的，所以，当我们需要对 UserDetails 进行缓存时，我们只需要定义一个 Ehcache 实例，然后把它注入给 EhCacheBasedUserCache 就可以了

## CachingUserDetailsService 配置实现

### 

![desc](media/bf2c3eb534c614d6c4025985546bc1bb.png)

# intercept-url配置

## 指定拦截的 url

![desc](media/0ce951b14b877646d085c88b8fa7734f.png)

## 指定访问权限通过 access 属性来指定 intercept-url 对应URL 访问所应当具有的权限

![desc](media/cf09376791f02db06f6c58fb7a1494c3.png)

### 

![desc](media/5040254c02aaceb22abde76219c2e322.png)

## 

![desc](media/61244aa6033f39fdfff931076538a7b5.png)

## 

![desc](media/5402423174e4d1f212d26c6f1fdfcb08.png)

# Filter

## Filter 顺序

![desc](media/c4d820e23656a3aee4b378ee00a6ff3d.png)

## FilterChain 中 Filter

### 

![desc](media/794cfe887e756b1e29e8872a84e3a185.png)

### Spring Security 对那些内置的 Filter 都指定了一个别名，同时指定了它们的位置。我们在定义 custom-filter 的 position、before 和 after 时使用的值就是对应着这些别名所处的位置。

## DelegatingFilterProxy

### 

![desc](media/f81b265a76acf8acd126ef9138500766.png)

## FilterChainProxy

### 

![desc](media/2d864ae6f52ce4404bdfdde900b9561a.png)

## 定义好的核心 Filter

### 

![desc](media/d9f05ddb7023bb35fa43d4181da8c8a4.png)

### FilterSecurityInterceptor

-   

    ![desc](media/3c94d0c1272b96960a1e8307177f5308.png)

### ExceptionTranslationFilter

-   自定义

    •

    ![desc](media/5927663ea54fca2494901f44df1f9371.png)

-   

    ![desc](media/1b22d7b397b7f55a4fb2214b12801a8f.png)

### SecurityContextPersistenceFilter

-   

    ![desc](media/4ae87b7d13e8a619af08d83933f48a97.png)

-   

    ![desc](media/9410785d8c9653654d574c8681125221.png)

### UsernamePasswordAuthenticationFilter

-   

    ![desc](media/1e69b5c816d5c20a5a93f79deb4b3a57.png)

-   

    ![desc](media/810313889d9a2840619729997bf85bc2.png)

# 退出登录 logout

## 

![desc](media/58cbfd93f85a175794a6f2434e649cc6.png)

# 匿名认证

## 

![desc](media/971676a1368d42a40e8ed2125f69d6b4.png)

## 配置

### 

![desc](media/44f1a12342ac85173d8406f7b672fc1c.png)

### 手动定义Bean

-   

    ![desc](media/82489b2fe948ee077096a5510ec657c6.png)

## AuthenticationTrustResolver

### 

![desc](media/59f9a601f4308284bff88566911c43e7.png)

# Remember-Me 功能

## 

![desc](media/51f3ca68f6bab6a76a297887b4c6deca.png)

## 基于简单加密 token 的方法

### 

![desc](media/71c10343d666f98b318e0001863979f6.png)

### 

![desc](media/3730eab308f51fbdfa4c874222736a40.png)

## 基于持久化 token 的方法

### 

![desc](media/bfaf392e772b18322520235939d6c757.png)

### 

![desc](media/c78c4ed97a51804bca1795abbef70ec7.png)

## Remember-Me 相关接口和实现类

### 

![desc](media/cef4162fc798cee254493261c3af8764.png)

## TokenBasedRememberMeServices

### 

![desc](media/6c4792a970a8118cac7686a0f1951c0d.png)

### 

![desc](media/022eac30696a59ff04e607cbbd5c1a07.png)

## PersistentTokenBasedRememberMeServices

### 

![desc](media/78f6117e43b72b5ee1949c23393fbbaf.png)

### 

![desc](media/5861f73c1bdf984e28ee8324153cac8a.png)

# session 管理

## Spring Security 通过 http 元素下的子元素 session-management 提供了对 Http Session 管理的支持。

## 检测 session 超时

### 

![desc](media/281193a972d38fe742d2dd8b7f994ad2.png)

## concurrency-control

### 

![desc](media/4cd83cd1f59d079385caf612f97be917.png)

### 

![desc](media/25811e036d760646164dcffa8cdef50b.png)

### 

![desc](media/0a2a7105d740ce3886355a0726aa59eb.png)

## session 固定攻击保护

### 

![desc](media/85dd3874966d4225770883f84f0fcc98.png)

# 权限鉴定基础

## 

![desc](media/735976581c542a1ebdcc61a70619365e.png)

## AOP Advice 思想

### 

![desc](media/5f59c08106be045af8ebf31642ce8aa0.png)

## AbstractSecurityInterceptor

### 

![desc](media/d7f91fd7be9edf91f7e5f27f26ff5daf.png)

## ConfigAttribute

### 

![desc](media/485172a584a83aa5ca5fe09e6bc8109f.png)

## RunAsManager

### 

![desc](media/44893d77215412bc4dde739763397c80.png)

## AfterInvocationManager

### 

![desc](media/45090353fc25e05f7d6fc8765de014fd.png)
