**Shiro**

# 简介

## 

![desc](media/f357a5dfdcfd9ea46df588ed4f802ba4.png)

### Authentication：身份认证 / 登录，验证用户是不是拥有相应的身份；Authorization：授权，即权限验证，验证某个已认证的用户是否拥有某个权限；即判断用户是否能做事情，常见的如：验证某个用户是否拥有某个角色。或者细粒度的验证某个用户对某个资源是否具有某个权限；Session Management：会话管理，即用户登录后就是一次会话，在没有退出之前，它的所有信息都在会话中；会话可以是普通 JavaSE 环境的，也可以是如 Web 环境的；Cryptography：加密，保护数据的安全性，如密码加密存储到数据库，而不是明文存储；Web Support：Web 支持，可以非常容易的集成到 Web 环境；Caching：缓存，比如用户登录后，其用户信息、拥有的角色 / 权限不必每次去查，这样可以提高效率；Concurrency：shiro 支持多线程应用的并发验证，即如在一个线程中开启另一个线程，能把权限自动传播过去；Testing：提供测试支持；Run As：允许一个用户假装为另一个用户（如果他们允许）的身份进行访问；Remember Me：记住我，这个是非常常见的功能，即一次登录后，下次再来的话不用登录了

## 

![desc](media/102f3646b6fbedd8b4591cc5666a91a0.png)

### Subject：主体，可以看到主体可以是任何可以与应用交互的 “用户”；SecurityManager：相当于 SpringMVC 中的 DispatcherServlet 或者 Struts2 中的 FilterDispatcher；是 Shiro 的心脏；所有具体的交互都通过 SecurityManager 进行控制；它管理着所有 Subject、且负责进行认证和授权、及会话、缓存的管理。Authenticator：认证器，负责主体认证的，这是一个扩展点，如果用户觉得 Shiro 默认的不好，可以自定义实现；其需要认证策略（Authentication Strategy），即什么情况下算用户认证通过了；Authorizer：授权器，或者访问控制器，用来决定主体是否有权限进行相应的操作；即控制着用户能访问应用中的哪些功能；Realm：可以有 1 个或多个 Realm，可以认为是安全实体数据源，即用于获取安全实体的；可以是 JDBC 实现，也可以是 LDAP 实现，或者内存实现等等；由用户提供；注意：Shiro 不知道你的用户 / 权限存储在哪及以何种格式存储；所以我们一般在应用中都需要实现自己的 Realm；SessionManager：如果写过 Servlet 就应该知道 Session 的概念，Session 呢需要有人去管理它的生命周期，这个组件就是 SessionManager；而 Shiro 并不仅仅可以用在 Web 环境，也可以用在如普通的 JavaSE 环境、EJB 等环境；所以呢，Shiro 就抽象了一个自己的 Session 来管理主体与应用之间交互的数据；这样的话，比如我们在 Web 环境用，刚开始是一台 Web 服务器；接着又上了台 EJB 服务器；这时想把两台服务器的会话数据放到一个地方，这个时候就可以实现自己的分布式会话（如把数据放到 Memcached 服务器）；SessionDAO：DAO 大家都用过，数据访问对象，用于会话的 CRUD，比如我们想把 Session 保存到数据库，那么可以实现自己的 SessionDAO，通过如 JDBC 写到数据库；比如想把 Session 放到 Memcached 中，可以实现自己的 Memcached SessionDAO；另外 SessionDAO 中可以使用 Cache 进行缓存，以提高性能；CacheManager：缓存控制器，来管理如用户、角色、权限等的缓存的；因为这些数据基本上很少去改变，放到缓存中后可以提高访问的性能Cryptography：密码模块，Shiro 提供了一些常见的加密组件用于如密码加密 / 解密的。

# 身份验证

## 在 shiro 中，用户需要提供 principals （身份）和 credentials（证明）给 shiro，从而应用能验证用户身份：principals：身份，即主体的标识属性，可以是任何东西，如用户名、邮箱等，唯一即可。一个主体可以有多个 principals，但只有一个 Primary principals，一般是用户名 / 密码 / 手机号。credentials：证明 / 凭证，即只有主体知道的安全值，如密码 / 数字证书等。最常见的 principals 和 credentials 组合就是用户名 / 密码了。接下来先进行一个基本的身份认证。另外两个相关的概念是之前提到的 Subject 及 Realm，分别是主体及验证主体的数据源。

## 环境准备

### 添加 junit、common-logging 及 shiro-core 依赖即可

## 登录 / 退出

### 使用 shiro.ini 配置文件，通过 [users] 指定了两个主体：zhang/123、wang/123。

### 

![desc](media/6968a584b7bda729a45f705990b7d95a.png)

## 身份认证流程

### 

![desc](media/4a2703d24ac97c159ad3933ac6c3e110.png)

### 

![desc](media/6c2a7e6a168c4eb998c10783967d22c9.png)

## Realm

### 

![desc](media/93385a371aebc78f4f6571b5593f8c88.png)

### 自定义 Realm 实现

-   

    ![desc](media/c4796c14b8ac3e128f1fccc325cb53b4.png)

### JDBC Realm 使用

-   依赖：使用 mysql 数据库及 druid 连接池
-   建三张数据库表：users（用户名 / 密码）、user_roles（用户 / 角色）、roles_permissions（角色 / 权限）
-   ini 配置（shiro-jdbc-realm.ini）

    •

    ![desc](media/aab144534795cc8905b0c33b5bc00855.png)

## Authenticator 及 AuthenticationStrategy

### 

![desc](media/c4de91ef3d6d9877ae22e41b0a3801fe.png)

### FirstSuccessfulStrategy：只要有一个 Realm 验证成功即可，只返回第一个 Realm 身份验证成功的认证信息，其他的忽略；AtLeastOneSuccessfulStrategy/：只要有一个 Realm 验证成功即可，返回所有 Realm 身份验证成功的认证信息；AllSuccessfulStrategy：所有 Realm 验证成功才算成功，且返回所有 Realm 身份验证成功的认证信息，如果有一个失败就失败了。ModularRealmAuthenticator 默认使用 AtLeastOneSuccessfulStrategy 策略。

### ini 配置文件 (shiro-authenticator-all-success.ini)

-   

    ![desc](media/14eef2a1e36cb83a5b6434da76a32de2.png)

# Shiro 授权

## 授权方式

### 

![desc](media/81515f0dbf94b969edc79007ecd00f48.png)

## 授权

### 基于角色的访问控制（隐式角色）

-   

    ![desc](media/20753eaf718277d7a8d07d33a8d087b2.png)

-   Shiro 提供了 hasRole/hasRoles 用于判断用户是否拥有某个角色/某些权限；
-   Shiro 提供的 checkRole/checkRoles 和 hasRole/hasAllRoles 不同的地方是它在判断为假的情况下会抛出 UnauthorizedException 异常。

### 基于资源的访问控制（显示角色）

-   Shiro 提供了 isPermitted 和 isPermittedAll 用于判断用户是否拥有某个权限或所有权限
-   

    ![desc](media/3e9ff7feb550c16572b32197c7829136.png)

## Permission

### 字符串通配符权限

-   

    ![desc](media/9340897173537a853154c2e3cd0f78d3.png)

-   

    ![desc](media/e0e28ec588c243c946d37dac5bf8a7b6.png)

### 授权流程

-   

    ![desc](media/65307653042a54fcf22c1b4e0f79dbde.png)

    •

    ![desc](media/b2c60a4f8d5f52ef94a9cc70bda01290.png)

## Authorizer、PermissionResolver及RolePermissionResolver

### SecurityManager 继承了 Authorizer 接口，且提供了 ModularRealmAuthorizer 用于多 Realm 时的授权匹配。PermissionResolver 用于解析权限字符串到 Permission 实例，而 RolePermissionResolver 用于根据角色解析相应的权限集合。

### ini 配置（shiro-authorizer.ini）

![desc](media/20b311e04fc36ffc244e9b4a11a43ce9.png)

-   设置 securityManager 的 realms 一定要放到最后，因为在调用 SecurityManager.setRealms 时会将 realms 设置给 authorizer，并为各个 Realm 设置 permissionResolver 和 rolePermissionResolver。另外，不能使用 IniSecurityManagerFactory 创建的 IniRealm，因为其初始化顺序的问题可能造成后续的初始化 Permission 造成影响。

### 定义 BitAndWildPermissionResolver 及 BitPermission

-   BitPermission 用于实现位移方式的权限，如规则是：权限字符串格式：+ 资源字符串 + 权限位 + 实例 ID；以 + 开头中间通过 + 分割；权限：0 表示所有权限；1 新增（二进制：0001）、2 修改（二进制：0010）、4 删除（二进制：0100）、8 查看（二进制：1000）；如 +user+10 表示对资源 user 拥有修改 / 查看权限。
-   Permission 接口提供了 boolean implies(Permission p) 方法用于判断权限匹配的；
-   BitAndWildPermissionResolver 实现了 PermissionResolver 接口，并根据权限字符串是否以 “+” 开头来解析权限字符串为 BitPermission 或 WildcardPermission。

### 定义 MyRolePermissionResolver

-   

    ![desc](media/673e0e9895d504972132842f824b5bfa.png)

### 自定义 Realm

-   

    ![desc](media/463cb76005049efb45b9cb8ef74bc5bf.png)

-   AuthorizingRealm 而不是实现 Realm 接口；推荐使用 AuthorizingRealm，因为： AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token)：表示获取身份验证信息；AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals)：表示根据用户身份获取授权信息。这种方式的好处是当只需要身份验证时只需要获取身份验证信息而不需要获取授权信息。

# Shiro InI 配置

## 根对象 SecurityManager

### Shiro 提供的 INI 配置也是非常类似的，即可以理解为是一个 IoC/DI 容器，但是区别在于它从一个根对象 securityManager 开始。

### java实现

![desc](media/0670a302b1a41d9e03696fc738c57273.png)

### 等价的 INI 配置（shiro-config.ini）

![desc](media/833af59ad60d5efbad7385971f7d178f.png)

### Shiro INI 配置中获取相应的 securityManager 实例

-   

    ![desc](media/6052491e5ced348634a25d2908e359d0.png)

## INI 配置

### ini 配置文件类似于 Java 中的 properties（key=value），不过提供了将 key/value 分类的特性，key 是每个部分不重复即可，而不是整个配置文件

### 

![desc](media/be17dd6af6ffc9b9a5d9d36140ebfb75.png)

### main] 部分提供了对根对象 securityManager 及其依赖对象的配置。

-   创建对象securityManager=org.apache.shiro.mgt.DefaultSecurityManager其构造器必须是 public 空参构造器，通过反射创建相应的实例。
-   常量值 setter 注入

    ![desc](media/7ff74fa4bac07fb577a5dcae2020c9e8.png)

-   对象引用 setter 注入

    ![desc](media/a539acf37871f6cd216e191a031b3453.png)

-   嵌套属性 setter 注入

    ![desc](media/dccb9466816f37b0df0c42a3055b09e6.png)

-   byte 数组 setter 注入

    ![desc](media/7c79dbd8a85c453d575b868432da477f.png)

-   Array/Set/List setter 注入

    ![desc](media/3492d40fd05cb06f6f1577b4128594ea.png)

-   Map setter 注入

    ![desc](media/4c276450598ef8a360dfc3192d447e16.png)

-   实例化 / 注入顺序后边的覆盖前边的注入。

### [users] 部分配置用户名 / 密码及其角色，格式：“用户名 = 密码，角色 1，角色 2”，角色部分可省略。

### [roles] 部分配置角色及权限之间的关系，格式：“角色 = 权限 1，权限 2”；

### [urls] 部分配置 url 及相应的拦截器之间的关系，格式：“url = 拦截器 [参数]，拦截器 [参数]

# 编码加密

## 编码 / 解码base64 和 16 进制字符串编码 / 解码

![desc](media/07514123dd89c8a09468574a229cc78f.png)

## 散列算法散列算法一般用于生成数据的摘要信息，是一种不可逆的算法，一般适合存储密码之类的数据，常见的散列算法如 MD5、SHA 等。一般进行散列时最好提供一个 salt（盐）

### 

![desc](media/4314709998b4a7a72459f24658e10672.png)

### Shiro 提供了 HashService，默认提供了 DefaultHashService 实现

-   

    ![desc](media/95c407a6a816bd60a1ff10f70b88cc71.png)

-   

    ![desc](media/1b2c48f32787975bebabaffc1a95161b.png)

## 加密 / 解密Shiro 还提供对称式加密 / 解密算法的支持，如 AES、Blowfish 等；当前还没有提供对非对称加密 / 解密算法支持

### AES 算法实现：

![desc](media/cb1bda40b4e758694950f1aad3627d9c.png)

## PasswordService/CredentialsMatcher提供加密密码及验证密码服务

### 

![desc](media/95ec5125589e54b7b837b63c71f4787a.png)

### DefaultPasswordService 配合 PasswordMatcher 实现简单的密码加密与验证服务

-   定义 Realm

    ![desc](media/b726ae65afea12bd2831442b5d8ccc70.png)

-   ini 配置（shiro-passwordservice.ini）

    ![desc](media/bb2f02f6bd98e6140bfa05196234d0ef.png)

    •

    ![desc](media/8aa2167176b49502bc647f3e89c92b7a.png)

### HashedCredentialsMatcher 实现密码验证服务

-   生成密码散列值

    ![desc](media/1a03af8a33775f3aaf5ed28155041c28.png)

-   生成 Realm

    •

    ![desc](media/0a59bbfd3d183273dd635bd44ec530c4.png)

    • JdbcRealm实现

    •

    ![desc](media/cc3ef0e5a2c3eb4248c3933c3f57d3d5.png)

-   ini 配置（shiro-hashedCredentialsMatcher.ini）

    •

    ![desc](media/348e0a6f47710090933cd50bcac60606.png)

    •

    ![desc](media/ef3762c3555b048c13401f3172e68d68.png)

### 密码重试次数限制

-   如在 1 个小时内密码最多重试 5 次，如果尝试次数超过 5 次就锁定 1 小时，1 小时后可再次重试，如果还是重试失败，可以锁定如 1 天，以此类推，防止密码被暴力破解。我们通过继承 HashedCredentialsMatcher，且使用 Ehcache 记录重试次数和超时时间。
-   

    ![desc](media/d0de6d7d070d2b2e4bef21b89e822c76.png)

# Realm

## 定义 Service 及 Dao

## UserRealm

![desc](media/8cb61f12390dced93d18ce46ce3be7dc.png)

## AuthenticationToken

![desc](media/119f78681cdac317f2360de2b7f881a1.png)

## AuthenticationInfo

![desc](media/b1eaeffbbb3e74504d431b1629e032a9.png)

## PrincipalCollection

### 

![desc](media/49559889d9511201c7003ac941bd099f.png)

## AuthorizationInfo

![desc](media/7d65e232599a4379ad7bdc12346570d9.png)

## SubjectSubject 是 Shiro 的核心对象，基本所有身份验证、授权都是通过 Subject 完成

### 身份信息获取

![desc](media/f0d7aebe22f8ac7a9e80ee6cc838139c.png)

### 身份验证

![desc](media/c9f1cd8f800c8c9322e6cc0180d34d9c.png)

### 角色授权验证

![desc](media/66aa7d94ffa819df8d81c1408536d5d5.png)

### 权限授权验证

![desc](media/89c5ee7d401eb5ae26f9600ded0250da.png)

### 会话

![desc](media/67b607839e9f9016b54b6b111ef128de.png)

### 退出void logout();

### RunAs

![desc](media/71cbb461a254ba6379be18cbb892a19c.png)

### 多线程

![desc](media/f9616f1112c02d58bd2c93d789395b93.png)

### Subject 的构建

-   

    ![desc](media/9e55855ad2a4d4f715183bdf532be00e.png)

-   自定义创建

    ![desc](media/3fc9254798ce35c864aff6397501d337.png)

### 使用

![desc](media/f3177547ca5e580202d108ea8d1dd97a.png)

# Web 集成

## 简介Shiro 提供了与 Web 集成的支持，其通过一个 ShiroFilter 入口来拦截需要安全控制的 URL，然后进行相应的控制，ShiroFilter 类似于如 Strut2/SpringMVC 这种 web 框架的前端控制器，其是安全控制的入口点，其负责读取配置（如 ini 配置文件），然后判断 URL 是否需要登录 / 权限等工作。

## 准备环境

### 依赖

-   

    ![desc](media/c2560bf3810f53bc3fa69e242f1b1c00.png)

-   

    ![desc](media/26543aca07eec2b3dc6737e30cfeeb11.png)

## ShiroFilter 入口

### 配置方式:web.xml

-   

    ![desc](media/347f232da47c655cc0ef8260e835355c.png)

### 与 Spring 集成

-   

    ![desc](media/2743c2c9d5211eee76c3a3422ee491cf.png)

## Web INI 配置

### 

![desc](media/cbf92b831562763e1f62d06168e1da22.png)

### Ant 风格模式

-   

    ![desc](media/bc1eb05f229fb84c962a0c5b5e533a00.png)

### 身份验证（登录）

-   登录 Servlet

    ![desc](media/55d926e8ad2c77be56fab74a9bea5c8e.png)

### 基于 Basic 的拦截器身份验证

-   shiro-basicfilterlogin.ini 配置

    ![desc](media/7711a786c945c879b7ce0d0a15ab438a.png)

-   web.xml把 shiroConfigLocations 改为 shiro-basicfilterlogin.ini 即可。

### 基于表单的拦截器身份验证

-   shiro-formfilterlogin.ini

    ![desc](media/8a1a7d5bef10f6350d84baf9c8dbfce9.png)

-   web.xml把 shiroConfigLocations 改为 shiro-formfilterlogin.ini 即可。
-   登录 Servlet

    ![desc](media/5dd409cee49f686a8ecee90e446b9143.png)

### 授权（角色 / 权限验证）

-   shiro.ini

    ![desc](media/fef2c07e100993215953e9e8e8b2556f.png)

-   web.xml把 shiroConfigLocations 改为 shiro.ini 即可。
-   RoleServlet/PermissionServlet

    ![desc](media/fdc134dc3a27424760402fdab36174f3.png)

### 退出

-   shiro.ini

    ![desc](media/490951839c089f2ba226898327a00f59.png)

-   LogoutServlet

    ![desc](media/c196e3d7d31e9aaf060834f8ac5f0748.png)

-   logout 拦截器用于退出

    ![desc](media/7eb5647fe66f17944c6fe072a3c64579.png)

# 拦截器机制

## 拦截器介绍

### 

![desc](media/98e2e10b5250ccdb47a35ad6f02e01c4.png)

### NameableFilterNameableFilter 给 Filter 起个名字，如果没有设置默认就是 FilterName

### OncePerRequestFilterOncePerRequestFilter 用于防止多次执行 Filter 的；也就是说一次请求只会走一次拦截器链；另外提供 enabled 属性，表示是否开启该拦截器实例，默认 enabled=true 表示开启，如果不想让某个拦截器工作，可以设置为 false 即可。

### ShiroFilterShiroFilter 是整个 Shiro 的入口点，用于拦截需要安全控制的请求进行处理

### AdviceFilterAdviceFilter 提供了 AOP 风格的支持，类似于 SpringMVC 中的 Interceptor：

![desc](media/836fa7df8ac793670f913eda9cf26aa2.png)

### PathMatchingFilterPathMatchingFilter 提供了基于 Ant 风格的请求路径匹配功能及拦截器参数解析的功能，如“roles[admin,user]”自动根据“，”分割解析到一个路径参数配置并绑定到相应的路径

![desc](media/be7a7b317a494499ae3b5ed5fb538610.png)

### AccessControlFilterAccessControlFilter 提供了访问控制的基础功能；比如是否允许访问/当访问拒绝时如何处理等

![desc](media/23d32a5b4f05a907f214d50da61bc56b.png)

-   

    ![desc](media/d5939605f101369135e29c05d079a590.png)

## 拦截器链

## 自定义拦截器

## 默认拦截器

# JSP 标签

## 导入标签库

![desc](media/ddaf9617b328ad0843f428a0cf92c617.png)

## guest 标签

![desc](media/1eb5ad934df3981f77885588f3eb7aa0.png)

## user 标签

![desc](media/154843b049a9696691ecf951aea07664.png)

## authenticated 标签

![desc](media/1893909f751099de5f74dbe4d9dd868c.png)

## notAuthenticated 标签

![desc](media/d1457714f9e066cdd01440a81e88259d.png)

## principal 标签

![desc](media/9b3b05c54b1a9e2a37f73446051a08b1.png)

## hasRole 标签

![desc](media/f201c0e535c6bf48fb40d949a6cc1892.png)

## hasAnyRoles 标签

![desc](media/f2308574512c56d058799dbe46173d56.png)

## lacksRole 标签

![desc](media/11ef33ee2da6a2caac0a1dd99ff0ba0c.png)

## hasPermission 标签

![desc](media/ea06bd30a14103a9e35234d3e41b60bc.png)

## lacksPermission 标签

![desc](media/1ae12029adb7629be431814923e115c5.png)

## 导入自定义标签库

![desc](media/6c7e808e46873f6070510656a2e97744.png)

# 会话管理

## Shiro 提供了完整的企业级会话管理功能，不依赖于底层容器（如 web 容器 tomcat），不管 JavaSE 还是 JavaEE 环境都可以使用，提供了会话管理、会话事件监听、会话存储 / 持久化、容器无关的集群、失效 / 过期支持、对 Web 的透明支持、SSO 单点登录的支持等特性。即直接使用 Shiro 的会话管理可以直接替换如 Web 容器的会话管理

## 会话

### 

![desc](media/06fd4b08b8f245106ae2d93523970336.png)

### session.getId();获取当前会话的唯一标识。session.getHost();获取当前 Subject 的主机地址，该地址是通过 HostAuthenticationToken.getHost() 提供的。

### session.getTimeout();session.setTimeout(毫秒);获取 / 设置当前 Session 的过期时间；如果不设置默认是会话管理器的全局过期时间。

### session.getStartTimestamp();session.getLastAccessTime();获取会话的启动时间及最后访问时间

### session.touch();如果是 JavaSE 应用需要自己定期调用 session.touch() 去更新最后访问时间；如果是 Web 应用，每次进入 ShiroFilter 都会自动调用 session.touch() 来更新最后访问时间。

### session.stop();更新会话最后访问时间及销毁会话；当 Subject.logout() 时会自动调用 stop 方法来销毁会话。如果在 web 中，调用 javax.servlet.http.HttpSession. invalidate() 也会自动调用 Shiro Session.stop 方法进行销毁 Shiro 的会话。

### 

![desc](media/73f4ec979e0c4e2ba1a3ef86b084b57c.png)

## 会话管理器

### 

![desc](media/d1faf3303b57531931b70cfcedfa6571.png)

### 

![desc](media/c072d2f748904dcdc5a68ad664a4a3cf.png)

### 

![desc](media/5b83546cd5bc6eeec1c02cbef9700c5c.png)

### 

![desc](media/3e2d335d1a3e7767f79948458a9c1873.png)

## 会话监听器

### 

![desc](media/bde4de46fec51aad62af77f2f4b7043e.png)

### 

![desc](media/4f4e25c00094ae9a427dd19af4069776.png)

## 会话存储 / 持久化

### 

![desc](media/e083f6edfd07335611bde355c1c5dcf1.png)

### 

![desc](media/2689d464d9b32919e03f68a294747991.png)

### 

![desc](media/b0978a9a176c10d78170226e27e82b85.png)

### 

![desc](media/bb18f39279e53865ffc81843896c39fc.png)

### 

![desc](media/36c0e8d1f3bad6d1db033843b94aefde.png)

## 会话验证

### 

![desc](media/9b61ab28657a77f85fd9431cc3de0409.png)

### 

![desc](media/a8750724baecd4b30cd47a6bcdaacc06.png)

### 

![desc](media/456b5f9d14e9612a8e9ce38e89f852b9.png)

## sessionFactory

### 

![desc](media/a47e56936de07e6cb191a695ced63b39.png)

### 

![desc](media/73d3373baba61933083c91eeee30d301.png)

# 缓存机制

## Shiro 提供了类似于 Spring 的 Cache 抽象，即 Shiro 本身不实现 Cache，但是对 Cache 进行了又抽象，方便更换不同的底层 Cache 实现。

### Shiro 提供的 Cache 接口：

![desc](media/417360057add0703d8db4a2ddc97de43.png)

### Shiro 提供的 CacheManager 接口：

![desc](media/764bb0711f6555b65fb9e4ff8d4d5130.png)

### CacheManagerAware 用于注入 CacheManager：

![desc](media/35266b113df55c28cbe8b06884357704.png)

## Realm 缓存

### Shiro 提供了 CachingRealm，其实现了 CacheManagerAware 接口，提供了缓存的一些基础实现；另外 AuthenticatingRealm 及 AuthorizingRealm 分别提供了对 AuthenticationInfo 和 AuthorizationInfo 信息的缓存。

### ini 配置

-   

    ![desc](media/c18c40120dabfd54e9128af90a0e88cd.png)

### 测试用例

-   

    ![desc](media/d8c53212ad828e2ed8cc6118dc6e8607.png)

## Session 缓存

### 

![desc](media/57ea64b876550b15f25117c45f5afe26.png)

# Spring 集成

## 

![desc](media/85b00c1148856077839ec1e3a655da8b.png)

## JavaSE 应用

### 

![desc](media/7d4a49e3240676c331ce3cfbeee373c5.png)

## Web 应用

### spring-shiro-web.xml

-   

    ![desc](media/75cda6ab9e4dbbbabfc7562a9e6cb799.png)

-   

    ![desc](media/3c77c199d59b88ea311d56674772e4d3.png)

### web.xml

-   

    ![desc](media/66e3b3fe90a5792422a3e910959f5e5e.png)

## Shiro 权限注解

### shiro 提供了相应的注解用于权限控制，如果使用这些注解就需要使用 AOP 的功能来进行判断，如 Spring AOP；Shiro 提供了 Spring AOP 集成用于权限注解的解析和验证。为了测试，此处使用了 Spring MVC 来测试 Shiro 注解，当然 Shiro 注解不仅仅可以在 web 环境使用，在独立的 JavaSE 中也是可以用的，此处只是以 web 为例了。

### 

![desc](media/84f5d44e5b40b80f9416e090119f67df.png)

### 权限注解

-   

    ![desc](media/0b9fcadb51d81e05ec8f258d60afd2ac.png)

# RememberMe

## 

![desc](media/17d73e2ff0c4264a7fe7bf998c26b048.png)

## RememberMe 配置

### spring-shiro-web.xml 配置

-   

    ![desc](media/277fd85ce3dcabe29bb30e4b9ad9adbb.png)

### 测试

-   

    ![desc](media/6901788813c607c1218d4e550a5e6389.png)

# SSL

## 生成数字证书

### 

![desc](media/cd8974e515119b7f190074e6d9b7f761.png)

## 设置 tomcat 下的 server.xml

### 

![desc](media/51665e4aeef8924946c6330f5bd1d5e4.png)

## 添加 SSL 到配置文件（spring-shiro-web.xml）

### 

![desc](media/4b9f46b1db7a8720cf675949aef3ca82.png)

## 测试

### 

![desc](media/0e432bfae0b27d875a1a8390df051071.png)

# 单点登录

## 

![desc](media/c8986c046eb2a74c24ed68c259957eb5.png)

## 服务器端

### 

![desc](media/bd4c0d30ac06a855b33e5157ef95c070.png)

## 客户端

### 准备

-   

    ![desc](media/8fe8716f12b369f1a99a66bb3ce3bedb.png)

### spring-shiro-web.xml 配置

-   

    ![desc](media/08e5a95e2ef553d2d2a9bdf7489eec34.png)

### 测试

-   

    ![desc](media/0f8f7a94c956bba419aa0dc0dd2c5380.png)

# 综合实例

## 

![desc](media/eaa2ad17cde4f933cc644fcc713ceb25.png)

## 简单数据字典

### 

![desc](media/9a4d1233e0fa59f9cd3ca47f84fa2584.png)

### 

![desc](media/24ebe78b32bbd117ddf056065361aac4.png)

## Service

### 

![desc](media/ba28dc1aeb770004bde3c0f3774c014e.png)

## UserRealm 实现

### 

![desc](media/fb40612300aabc432bcf9259e63277e1.png)

## Web 层控制器

### 

![desc](media/515ed29ace5e5ea1fa7a32e62cd7604d.png)

## Web 层标签库

### com.github.zhangkaitao.shiro.chapter16.web.taglib.Functions 提供了函数标签实现，有根据编号显示资源 / 角色 / 组织机构名称，其定义放在 src/main/webapp/tld/zhang-functions.tld。

### 

![desc](media/a929afb4be4470edc25341adffe2c028.png)

## Web 层异常处理器

### 

![desc](media/4b0b2036ccae64dbac115d9f40cd1855.png)

## Spring 配置

### spring-config.xml

-   定义了 context:component-scan 来扫描除 web 层的组件、dataSource（数据源）、事务管理器及事务切面等

### spring-config-cache.xml

-   定义了 spring 通用 cache，使用 ehcache 实现

### spring-config-shiro.xml

-   定义了 shiro 相关组件
-   

    ![desc](media/754e046b04baff355b4c11ba1ea1901b.png)

### spring-mvc.xml

-   定义了 spring mvc 相关组件。
-   

    ![desc](media/73a9a517402b6e00edb44754d95e9d93.png)

### spring-mvc-shiro.xml

-   定义了 spring mvc 相关组件。
-   

    ![desc](media/6b5ae6842339426da68d69fd5b907607.png)

### web.xml 配置文件

-   定义 Spring ROOT 上下文加载器、ShiroFilter、及 SpringMVC 拦截器。

## JSP 页面

### 

![desc](media/52d96b7545ffe04820fca574b139341f.png)

# OAuth2

## OAuth 角色

### 

![desc](media/0ea04e8cbfde51fd62583a21ca65a5e4.png)

## 服务器端

### POM 依赖

-   

    ![desc](media/02e49ec92caf902f812e7acddfc4a4b9.png)

### 数据字典

-   

    ![desc](media/1c509ad502d61723598b95b4e7e4caf4.png)

### Service

-   

    ![desc](media/c292e23405240deacfd017c155f513e2.png)

### 后端数据维护控制器

-   具体请参考 com.github.zhangkaitao.shiro.chapter17.web.controller 包下的 IndexController、LoginController、UserController 和 ClientController，其用于维护后端的数据，如用户及客户端数据；即相当于后台管理。
-   授权控制器 AuthorizeController

    •

    ![desc](media/8df3a48c0da642dc9684714e9d311689.png)

    •

    ![desc](media/7a8c7a3ae361099f931ceed12ed86235.png)

-   访问令牌控制器 AccessTokenController

    •

    ![desc](media/c059d0b9eb0a085332663ad9245b5559.png)

    •

    ![desc](media/1070ebebf61b08ad9cd26088d743197b.png)

-   资源控制器 UserInfoController

    •

    ![desc](media/71cc8d3bd3b019327a3c7a097cdf3e56.png)

    •

    ![desc](media/1fab0b864c99b10bd876013157d56494.png)

### Spring 配置文件

-   

    ![desc](media/2ac5a41dc6e4a1f4aa6ecb386818db55.png)

### 服务器维护

-   

    ![desc](media/d1dfd86467725f9991e8661e3b08bca9.png)

## 客户端

### 客户端流程：如果需要登录首先跳到 oauth2 服务端进行登录授权，成功后服务端返回 auth code，然后客户端使用 auth code 去服务器端换取 access token，最好根据 access token 获取用户信息进行客户端的登录绑定。这个可以参照如很多网站的新浪微博登录功能，或其他的第三方帐号登录功能。

### POM 依赖

-   

    ![desc](media/f88db476f12f75c3613387114a567761.png)

### OAuth2Token

-   

    ![desc](media/3c8fdf7d36ab8c724f0a71c41c502f90.png)

### OAuth2AuthenticationFilter

-   

    ![desc](media/7d65152a07de32e86dec1d2a9a78156d.png)

-   

    ![desc](media/4570a0d9774582aa7428111557ed1fc2.png)

### OAuth2Realm

-   

    ![desc](media/7e9d10191b573857e56b630f2990e76c.png)

### Spring shiro 配置（spring-config-shiro.xml）

-   

    ![desc](media/49bc4f004e0d518f04969ed488e453d8.png)

## 测试

### 

![desc](media/7b1ea36e172899b5251df7f2a2db0784.png)

# 并发登录控制

## 并发登录人数控制

### 

![desc](media/b9f5a9822e8d15763757c99b58460302.png)

## spring-config-shiro.xml

### 

![desc](media/d9878afb48b8c4e89af66b0e72be6ed0.png)

## 测试

### 

![desc](media/1765ee3c1ea3da98ac1cde2bbbd6e23a.png)

# 动态 URL

## 用过 Spring Security 的朋友应该比较熟悉对 URL 进行全局的权限控制，即访问 URL 时进行权限匹配；如果没有权限直接跳到相应的错误页面。Shiro 也支持类似的机制，不过需要稍微改造下来满足实际需求。不过在 Shiro 中，更多的是通过 AOP 进行分散的权限控制，即方法级别的；而通过 URL 进行权限控制是一种集中的权限控制。本章将介绍如何在 Shiro 中完成动态 URL 权限控制。

## 实体

### 

![desc](media/2e306b4a957b6749ce2c14bec8f47966.png)

## Service

### 

![desc](media/4df54c214cb46074d942395cdc0a2b1f.png)

## ShiroFilerChainManager

### 

![desc](media/cde416640814300e0d29449857039851.png)

### Shiro 拦截器的流程

-   

    ![desc](media/1fec15d9cb3a94b1cbffdbc6e16dca24.png)

### FilterChainManager 接口：

-   

    ![desc](media/e57a163892e443192abaa69a0414c54a.png)

## CustomPathMatchingFilterChainResolver

### 

![desc](media/c6395b4d0593422eca52d6ac563e58de.png)

## CustomDefaultFilterChainManager

### 

![desc](media/a1fc0c561bd9f872226c7885f444730e.png)

## Spring 配置——spring-config-shiro.xml

### 

![desc](media/2ca63eb396816d08c9a1314427753b1f.png)

# 无状态 Web

## 无状态 Web 应用集成

### 

![desc](media/b3af595bede721329de5a879f9b2cc6e.png)

## 服务器端对于服务器端，不生成会话，而是每次请求时带上用户身份进行认证。

### 服务控制器

-   

    ![desc](media/80f64002f0ec091c638bcbc62796d821.png)

### 加密工具类

-   

    ![desc](media/92d0306e0dc5dc25a3738069425efb4e.png)

### Subject 工厂

-   

    ![desc](media/24dc514645152940ecfe3ca3c1900cfc.png)

### StatelessAuthcFilter

-   

    ![desc](media/878099add34a2748d84a404f8866efd0.png)

### StatelessToken

-   

    ![desc](media/0a3b09145752751f58335c048b78e978.png)

### StatelessRealm

-   

    ![desc](media/827ed2fd633f3c9f67351a317fea541d.png)

### spring-config-shiro.xml

-   

    ![desc](media/c74fafa8e007b311a4b49d049f08de56.png)

## 客户端

### 启动服务器

-   

    ![desc](media/d21df570c48b7a141e0bfeaa31e52c60.png)

### 测试成功情况

-   

    ![desc](media/86b968717b30585dc720de5f1bf85108.png)

### 测试失败情况

-   

    ![desc](media/bd3c7bc84adfa1cdc0b6bd0fd3e2b7f8.png)

# 授予身份和切换身份

## 实体

### 

![desc](media/a0e8fa11d8470056751d2109f31761de.png)

## Service

### 

![desc](media/31a2c01a78ba7ca909a64383852114e3.png)

## Web 控制器 RunAsController

### 身份列表

-   

    ![desc](media/4c94c81f197e92fae804283cee66d3e5.png)

### 授予身份

-   

    ![desc](media/d792b0943dbad9fea705c91899c8f0a4.png)

### 回收身份

-   

    ![desc](media/af41e5f2694f78e7a85e8be799c0b4cf.png)

### 切换身份功能

-   

    ![desc](media/d00dbe61d6a1d168f9b0f09880456f0f.png)

### 切换到上一个身份

-   

    ![desc](media/51db4e48adc004c775a0dfc15e713b25.png)

# 集成验证码

## 添加 JCaptcha 依赖

### 

![desc](media/9f193e994da8362633b72870673eca3e.png)

## GMailEngine

## MyManageableImageCaptchaService

### 

![desc](media/a769ba21012d8113c91f6948dda33966.png)

## JCaptcha 工具类

### 

![desc](media/2ebf2254e0725b8e2cbafc4e4e941ff5.png)

## JCaptchaFilter

### 

![desc](media/4d9295478eedbe0af004e00221a82141.png)

## JCaptchaValidateFilter

### 

![desc](media/a626e548e7e9614473afd307c9932e1b.png)

## MyFormAuthenticationFilter

### 

![desc](media/1a830254ed59629a901f830eacc175d8.png)

## spring-config-shiro.xml

### 

![desc](media/00dbdbc68d7639b74b1cb05a584e2796.png)

## login.jsp 登录页面

### 

![desc](media/44226938a99eaca0ea95047f7e0f0bce.png)

# 多项目

## 部署架构

### 

![desc](media/5a0a2fb7440cf87556068fb4252d15e2.png)

## 项目架构

### 

![desc](media/6b70c5c26a0e702b68ae44d7f5c97347.png)

## 模块关系依赖

### 

![desc](media/f159491f8a107e701919296698787903.png)

## shiro-example-chapter23-pom 模块

### 

![desc](media/e9dd9e06d36d3d049e618fd7876d6162.png)

## shiro-example-chapter23-core 模块

### 

![desc](media/416f31bb44a2d5f3b7d6f8dfcea284ae.png)

## shiro-example-chapter23-server 模块

### 

![desc](media/a136b7228416831daba9e9ffba6d6a6c.png)

### Service

-   

    ![desc](media/f40d4304aa08ae48700445627489845c.png)

### UserRealm

-   

    ![desc](media/83166bd6bcbfd31c4b99473ea2cb8636.png)

### ServerFormAuthenticationFilter

-   

    ![desc](media/ee300dfaddc2637f9308a80f6b38d88a.png)

### session

-   

    ![desc](media/4e7ec4ed5019187736fb5ca3c20a22a4.png)

### RemoteService

-   

    ![desc](media/0c9603788f4c2ddbc4cdfc2b86ab3281.png)

### spring-config-shiro.xml

-   

    ![desc](media/7c5abe74dce018fef933d4aac7429688.png)

### 服务器端数据维护

-   

    ![desc](media/8347dd944df6fcd4eafb2a988a7b46ae.png)

## shiro-example-chapter23-client 模块

### spring-client-remote-service.xml

-   

    ![desc](media/b46923dd24acee0ff889beec14982277.png)

### ClientRealm

-   

    ![desc](media/08b6d57dc5f5e725342caafc72a8b885.png)

### ClientSessionDAO

-   

    ![desc](media/66d06bdef12961b9159d1b5cc7d3c31c.png)

### ClientAuthenticationFilter

-   

    ![desc](media/8af21f80459c9f4a0c5585dfc7b0749f.png)

### ClientShiroFilterFactoryBean

-   

    ![desc](media/11cb4b59bc1d6d0f9a09ac4e7f6f069c.png)

### spring-client.xml

-   

    ![desc](media/d427f2adaea6251aacc86df8d1beb5d0.png)

### client/shiro-client-default.properties

-   

    ![desc](media/09890cc106331b77237ce42e6fb93953.png)

## shiro-example-chapter23-app \* 模块

### 依赖

-   

    ![desc](media/04d4d298a6ff12eed0de14ee92b0a823.png)

### client/shiro-client.properties

-   

    ![desc](media/f696de5d1765c0f2dd6258f7ee093a6d.png)

### web.xml

-   

    ![desc](media/668bd034bdf9d027c053c5e18191d864.png)

### 控制器

-   

    ![desc](media/f4664a85e3964f4fcaaf21dc4140b151.png)

## 测试

### 安装配置启动 nginx

-   

    ![desc](media/daf3542ab2d3f5b0ae6817f88c8c6ea9.png)

### 安装依赖

-   

    ![desc](media/a16b7d443f750a42f79da566caabceb9.png)

### 启动 Server 模块

-   

    ![desc](media/22019f5d011958258389bef3c4a92acd.png)

### 启动 App\* 模块

-   

    ![desc](media/94ef84b2ce334cd380b76d8a52e96f6a.png)

### 服务器端维护

-   

    ![desc](media/7a8db50c459d57b17872ffd65513604f.png)

### App 模块身份认证及授权\*\*

-   

    ![desc](media/8aa6337ccfe2078e3f8fbc1ecdc75e51.png)

## 本示例缺点

### 

![desc](media/73838ead32c62cb1b93e8ae9530e321c.png)

# 在线会话管理

## 会话控制器

### 

![desc](media/1130db710a5e2f591a400c0b1eefc5c1.png)

### 

![desc](media/04b4e7c8e74c24744be3214a3039888e.png)

## ForceLogoutFilter

### 

![desc](media/03d69d777036ed3463be7447346006c5.png)

## 登录控制器

### 

![desc](media/7f8944270091b337a4365e6faa23c3bd.png)

## spring-config-shiro.xml

### 

![desc](media/c13a5babc1e9eb9b0cf4ea64abd95766.png)
