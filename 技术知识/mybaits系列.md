**mybaits系列**

# mybaits

## MyBatis特性

### 定制化sql、存储过程、高级映射

### 避免JDBC和手动设置参数

### 使用简单的XML或注解用于配置和原始映射

### 半自动的ORM框架

## 搭建MyBatis

### 引入依赖

-   

    ![desc](media/963bb24c98c3dad04657e27240b7ac04.png)

### 创建MyBatis的核心配置文件

-   mybatis-config.xml

    ![desc](media/0f13ed58233473289c4f4db0caafaa4d.png)

### 创建mapper接口

### 创建MyBatis的映射文件

-   ORM（Object Relationship Mapping）对象关系映射。

    ![desc](media/ea70c6a1bdf92554b89f432bf7081ee5.png)

-   

    ![desc](media/53611bdb97b0f2d80d413892bfcc4848.png)

### 通过junit测试功能

-   

    ![desc](media/13d4b19ffacf5cd0dddd1e14b6b4f798.png)

-   

    ![desc](media/4f3b2d74b2b6276d3ad7f67557bc0b2a.png)

### 加入log4j日志功能

-   加入依赖

    •

    ![desc](media/4e8d5506a035e29e1f88c415a75d088e.png)

-   加入log4j的配置文件

    •

    ![desc](media/17db6a8af66fb40d4855b4ba653665f6.png)

    • 日志的级别FATAL(致命)\>ERROR(错误)\>WARN(警告)\>INFO(信息)\>DEBUG(调试)从左到右打印的内容越来越详细

## 核心配置文件详解

### 核心配置文件中的标签必须按照固定的顺序：properties?,settings?,typeAliases?,typeHandlers?,objectFactory?,objectWrapperFactory?,reflectorFactory?,plugins?,environments?,databaseIdProvider?,mappers?

### 

![desc](media/32fe91f733b40f9fef2012948f55bd0d.png)

-   

    ![desc](media/6c1e0d1633320b61d7a7edc09b633427.png)

## MyBatis的增删改查

### insert, delete , update , select

### 

![desc](media/c6a6ca832b94321a770da6ad60213dcc.png)

## MyBatis获取参数值的两种方式

### 

![desc](media/2f820b0549f1f7ab18ea074abfb6ef10.png)

## MyBatis的各种查询功能

### 查询多条数据为map集合

![desc](media/a8a0b99824d61934c02ba97b70a6899e.png)

## 特殊SQL的执行

### 模糊查询(like)

### 批量删除(in)

### 动态设置表名(传参）

### 添加功能获取自增的主键

-   

    ![desc](media/81c884dcabd9ac9edba200975be03168.png)

## 自定义映射resultMap

### resultMap处理字段和属性的映射关系

-   

    ![desc](media/a0538c0b35796e83cfeb19bacd75a2f7.png)

### 多对一映射处理

-   级联方式处理映射关系

    •

    ![desc](media/832c82332c350916175a5260a10d2f6c.png)

-   使用association处理映射关系

    •

    ![desc](media/f33487ebdacd043f8c89cfed04baecd0.png)

-   分步查询

    •

    ![desc](media/b2649374a16fd5a738499e335016be2d.png)

    •

    ![desc](media/417444fc9fccba251ecdbd515cbd854f.png)

### 一对多映射处理

-   collection

    •

    ![desc](media/2c9d05600e9651a45f9d3139794c0637.png)

-   分步查询

    •

    ![desc](media/a60a584ed8c2917b250cd0ce82b10607.png)

## 动态SQL

### if：if标签可通过test属性的表达式进行判断，若表达式的结果为true，则标签中的内容会执行；反之标签中的内容不会执行\<if test="age != '' and age != null"\>and age = \#{age}\</if\>

### where

-   

    ![desc](media/415f040f26b6867f6e15e2af23ce790b.png)

-   

    ![desc](media/8c9780b3326bd3f7e1db7e55ae33fa87.png)

### trim

-   

    ![desc](media/e992fb7227e2a7ec85aae908d0ba287d.png)

-   

    ![desc](media/fb0821f9473a6bd1ebc268c51face0c7.png)

### choose、when、otherwise

-   

    ![desc](media/c43f6c3fa10beea63b5f2b30b579b3de.png)

### foreach

-   

    ![desc](media/867e72e16b4b332e3dee6ad5ec5ea1c6.png)

-   

    ![desc](media/a5b9b2827252666de11c1002f1e69b33.png)

### SQL片段

-   

    ![desc](media/3993b971fa1ed7c4dabe9de102662ce3.png)

## MyBatis的缓存

### MyBatis的一级缓存

-   

    ![desc](media/764dae9dd162ad9a58c7909cf867d575.png)

### 二级缓存

-   

    ![desc](media/8a602a9ae4c247e245476eb62020c17c.png)

-   二级缓存的相关配置

    •

    ![desc](media/cb21a7458f958135eeb5227cbeb091c8.png)

### 缓存查询的顺序

-   1、先查询二级缓存，因为二级缓存中可能会有其他程序已经查出来的数据，可以拿来直接使用。2、如果二级缓存没有命中，再查询一级缓存3、如果一级缓存也没有命中，则查询数据库4、SqlSession关闭之后，一级缓存中的数据会写入二级缓存

### 整合第三方缓存EHCache

-   添加依赖

    •

    ![desc](media/735407821a2d759e5682f0370c062f25.png)

-   各jar包功能

    •

    ![desc](media/49d90bdd7b81fa771ab4ce1221085cc8.png)

-   创建EHCache的配置文件ehcache.xml

    •

    ![desc](media/642c9e4f738ab8be417d27a6170f296c.png)

-   设置二级缓存的类型

    • \<cache type="org.mybatis.caches.ehcache.EhcacheCache"/\>

-   加入logback日志

    • 存在SLF4J时，作为简易日志的log4j将失效，此时我们需要借助SLF4J的具体实现logback来打印日志。创建logback的配置文件logback.xml

    •

    ![desc](media/b03a5738f32dd4fdd8ce3936fe46ed20.png)

-   EHCache配置文件说明

    •

    ![desc](media/e0471fcf2815c28ff705dbf38863cd2e.png)

## 逆向工程

### 

![desc](media/e08cc4f140310feaf1a5b3f1d2c7d2a0.png)

### 创建逆向工程

-   添加依赖和插件

    •

    ![desc](media/009b52a3dc606cfa964b85589f2dfad9.png)

-   MyBatis的核心配置文件
-   逆向工程的配置文件

    • 文件名必须是：generatorConfig.xml

    ![desc](media/a1d2c88ce7aa4876de32bd105f127b3c.png)

-   执行MBG插件的generate目标

    •

    ![desc](media/6b4a2ef3b87b43231107af3a472f8c2a.png)

### QBC查询

-   

    ![desc](media/9b6a53034844b3ca9958af797a29206e.png)

## 分页插件

### 插件配置

-   添加依赖

    •

    ![desc](media/474bd4205677c81d8f45386506d39a77.png)

-   配置分页插件

    • 在MyBatis的核心配置文件中配置插件

    ![desc](media/8d0aeaf856d67a2981c5c062743b00cb.png)

### 使用

-   a\>在查询功能之前使用PageHelper.startPage(int pageNum, int pageSize)开启分页功能
-   b\>在查询获取list集合之后，使用PageInfo\<T\> pageInfo = new PageInfo\<\>(List\<T\> list, intnavigatePages)获取分页相关数据
-   c\>分页相关数据

    •

    ![desc](media/c87f3506ef3c72073068662501674090.png)

# mybaits_plus

## 简介

### mybaits的增强工具

### 特性

-   无侵入、损耗小（直接面向对象操作）、强大的CRUD、支持lambda、支持主键自动生成（内含分布式唯一 ID 生成器 - Sequence）、支持 ActiveRecord 模式（继承model类）、支持自定义全局通用操作、内置代码生成器、内置分页插件、内置性能分析插件、内置全局拦截插件

## 配置使用

### spring boot配置

-   引入依赖

    • \<dependency\>\<groupId\>com.baomidou\</groupId\>\<artifactId\>mybatis-plus-boot-starter\</artifactId\>\<version\>3.5.1\</version\>\</dependency\>\<dependency\>\<groupId\>org.projectlombok\</groupId\>\<artifactId\>lombok\</artifactId\>\<optional\>true\</optional\>\</dependency\>

-   配置application.yml

    • 不用额外配置

    • 添加日志

    • \# 配置MyBatis日志，控制台显示mybatis-plus:configuration:log-impl: org.apache.ibatis.logging.stdout.StdOutImpl

-   在Spring Boot启动类中添加@MapperScan注解，扫描mapper包

### spring配置

-   引入依赖

    • spring-context、spring-jdbc、spring-test、druid、junit、mysql-connector-java、slf4j-api、logback-classic、lombok、mybatis-plus

    •

    ![desc](media/4709775fa41e7dcb424683862c024a09.png)

-   创建MyBatis的核心配置文件

    • 在resources下创建mybatis-config.xml

    •

    ![desc](media/3e069aab6ede3f1c3d8302680cb2637c.png)

-   相关mapper及xml文件的设置
-   创建jdbc.properties
-   创建Spring的配置文件

    • 在resources下创建applicationContext.xml

    • 引入jdbc.properties

    • 配置Druid数据源

    • - 配置用于创建SqlSessionFactory的工厂bean

    • 设置MyBatis配置文件的路径（可以不设置）

    • 设置数据源

    • 设置类型别名所对应的包（实体类）

    • 设置映射文件的路径

    • 配置mapper接口的扫描配置

    •

    ![desc](media/4a4710eec7133a9bcfb74eca3b933c8e.png)

-   添加日志功能

    • 在resources下创建logback.xml

    • 定义日志文件的存储地址 name="LOG_HOME"

    • 控制台日志， 控制台输出

    • myibatis log configure

    • 日志输出级别

    •

    ![desc](media/08eca7525cf2a0c7ef8058971f565541.png)

-   测试

    • 通过IOC容器

    • ApplicationContext ac = newClassPathXmlApplicationContext("applicationContext.xml");

    • Spring整合junit

    • //在Spring的环境中进行测试@RunWith(SpringJUnit4ClassRunner.class)//指定Spring的配置文件@ContextConfiguration("classpath:applicationContext.xml")

-   加入MyBatis-Plus

    • 修改applicationContext.xml

    •

    ![desc](media/fc9d971e40159b1cbc5b0302b7aa31dd.png)

    • \<beanclass="com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean"\>

## 基本CRUD

### BaseMapper

### IService

## 常用注解

### @TableName（”表名“）

-   Springboot的全局配置

    •

    ![desc](media/c5ec9475ad34d51bd003a53cc518725e.png)

-   Spring

    •

    ![desc](media/d4408853524d13c2152f4f5316f3a9ae.png)

### @TableId（”主键名“）识别主键

-   常用的主键策略：

    •

    ![desc](media/78a199f1169dd4356313e19f9153b523.png)

-   配置全局主键

    • springboot

    •

    ![desc](media/9435e2bf27ffb8411fe428a77857ac44.png)

    • spring

    •

    ![desc](media/b799036e48f9d0ee54d8b62410662fc0.png)

-   雪花算法

    • 数据库垂直分表

    • 垂直分表适合将表中某些不常用且占了大量空间的列拆分出去

    • 水平分表

    • 水平分表适合表行数特别大的表，针对数目进行分表

    • 全局唯一的数据id的处理

    • 主键自增

    • 优点：可以随着数据的增加平滑地扩充新的表。例如，现在的用户是 100 万，如果增加到 1000 万，只需要增加新的表就可以了，原有的数据不需要动。

    • 缺点：分布不均匀。假如按照 1000 万来进行分表，有可能某个分段实际存储的数据量只有 1 条，而另外一个分段实际存储的数据量有 1000 万条。

    • 取模

    • 复杂点：初始表数量的确定。表数量太多维护比较麻烦，表数量太少又可能导致单表性能存在问题。优点：表分布比较均匀。缺点：扩充新的表很麻烦，所有数据都要重分布。

    • 雪花算法

    • 长度共64bit（一个long型）。

    •

    ![desc](media/35fbbd4a6acc957050eb022814321494.png)

    • 优点：整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞，并且效率较高。

### @TableField

-   在实体类属性上使@TableField("username")设置属性所对应的字段名

### @TableLogic：逻辑删除

## 条件构造器和接口

### 

![desc](media/1139ea28e0f1198ddbb724e25bf76ef9.png)

### 结合sql语句进行优先级排序

### 关系式

-   

    ![desc](media/9979328a51e8338a2a5b26116676ebac.png)

### condition:参数条件判断

-   query.like(StringUtils.isNotBlank(name), Entity::getName, name) .eq(age!=null && age \>= 0, Entity::getAge, age)

## 插件

### 分页插件

-   配置

    • springboot

    •

    ![desc](media/59a5cc92e591a4e1a12e0f73511f77d6.png)

    • spring

    •

    ![desc](media/c6a38290382c5e87b796b9f7efd8690e.png)

### xml自定义分页

-   springboot

    • XXXmapper定义

    ![desc](media/2a6636947cbc4fa283f46844b46cd5bb.png)

    • XXXmapper.xml的sql

    ![desc](media/a2537b6fa13b87d1074ae43b07c99d57.png)

### 乐观锁

-   实现流程

    •

    ![desc](media/39361e56cac2940cbbf55bb611f15b92.png)

-   Mybatis-Plus实现乐观锁

    • springboot

    • 实体类声明@Version

    •

    ![desc](media/9af213c1671096e6429d1130db341c9b.png)

    • 添加乐观锁插件配置

    •

    ![desc](media/939d22de3a13828a7cd9fdfac6f6e3d9.png)

    • spring

    •

    ![desc](media/8c69997a56c3dafe3cfc8e48005a5776.png)

## 通用枚举

### 创建通用枚举类型

-   

    ![desc](media/ca786ca1cb25a053e5aae3e991c282dc.png)

### 配置扫描通用枚举

-   springboot

    •

    ![desc](media/bc93c7b34ad46a3b3738460c48003b12.png)

-   spring

    •

    ![desc](media/0e1a655b5b855d0c25a5c90b1bcdb269.png)

## 代码生成器

### 引入依赖

-   

    ![desc](media/7d060eefb971809711b6a9ba26ec43df.png)

### 快速生成

-   

    ![desc](media/dd1def8f11fe83b3b05987f82dec7033.png)

## 多数据源

### 引入依赖

-   

    ![desc](media/d87b5fd35601a359cdab79695dcbb70c.png)

### 配置多数据源

-   

    ![desc](media/14636d1f28088e3134194716233c6b90.png)

### 在XXXServiceImpl中使用 @DS("master") //指定所操作的数据源
