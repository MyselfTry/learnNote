**NGINX分析**

# 基础知识点

## 特点

### 

![desc](media/0a8b1c601ad33fe6c74f7cd1065ea0d0.png)

## 架构

### 

![desc](media/4a585e554a73f9c50bdf12e553cc368a.png)

-   

    ![desc](media/b05c6d17aa7dd44366f48a26236ed137.png)

### 管理通信和worker进程

-   kill -HUP pid

    • 告诉 Nginx，从容地重启 Nginx，首先 master 进程在接到信号后，会先重新加载配置文件，然后再启动新的 worker 进程，并向所有老的 worker 进程发送信号，告诉他们可以光荣退休了。新的 worker 在启动后，就开始接收新的请求，而老的 worker 在收到来自 master 的信号后，就不再接收新的请求，并且在当前进程中的所有未处理完的请求处理完成后，再退出。

-   ./nginx -s reload，就是来重启 Nginx，./nginx -s stop

    • 新的 Nginx 进程在解析到 reload 参数后，就知道我们的目的是控制 Nginx 来重新加载配置文件了，它会向 master 进程发送信号，然后接下来的动作，就和我们直接向 master 进程发送信号一样了

### 进程模型

-   

    ![desc](media/2a049d1af39339918859e5f044417e87.png)

-   

    ![desc](media/2b8b8e1dfee9e51b98d4d7e0888cfb50.png)

-   异步非阻塞的方式来处理
-   

    ![desc](media/1838d63dd5bf823ab88156015155a9d6.png)

### Nginx 的事件处理模型

-   

    ![desc](media/84a732cccc932df1b6389f31840eef23.png)

-   处理信号与定时器

    •

    ![desc](media/8e4d0b20d58b55c4c86c27eabd597749.png)

## 基础概念

### connection

-   

    ![desc](media/8d58f2fc88d6d79983fc9c4238919014.png)

-   

    ![desc](media/bf2a4ac577f8d0ce1718a9deaeaa0cd6.png)

### request

-   

    ![desc](media/7cfc84859f392ed2c09485db1bbed91f.png)

-   

    ![desc](media/0511117c24adaee2746ba79882088b7d.png)

### keepalive

-   

    ![desc](media/79e588d8277195de4d28155921b13b1e.png)

### pipe

-   

    ![desc](media/2cf65a1dcccd1aebed76053838ca75b5.png)

### lingering_close

-   

    ![desc](media/5d9f3af2bb83d6a7e156a062a66a2635.png)

### w3cschool.cn-Nginx 基础概念.pdf

## 基本数据结构

### ngx_str_t

### ngx_pool_t

### ngx_array_t

### ngx_hash_t

### ngx_hash_wildcard_t

### ngx_hash_combined_t

### ngx_hash_keys_arrays_t

### ngx_chain_t

### ngx_buf_t

### ngx_list_t

### ngx_queue_t

### w3cschool.cn-Nginx 基本数据结构.pdf

## 配置系统

### 

![desc](media/ea89d5a3595036563b07af8ae25cc496.png)

### 指令概述配置指令是一个字符串，可以用单引号或者双引号括起来，也可以不括。但是如果配置指令包含空格，一定要引起来。

### 指令参数

-   

    ![desc](media/542de59f43c7a96f6d7f12d7aba4a207.png)

### 指令上下文

-   

    ![desc](media/75761519c07f47cc0d49c98f76d01ffd.png)

-   示例

    •

    ![desc](media/f329bdac984d3a4b986f7a7519c28e36.png)

### 模块化体系结构

-   

    ![desc](media/4af2d943fdf8804adf433d7254e68bf9.png)

-   模块概述

    •

    ![desc](media/3bc57ab59c2999e24d4bf92f28e78770.png)

-   模块的分类

    •

    ![desc](media/763593b907798d321ee1eca83bb16e28.png)

## 请求处理

### 

![desc](media/8f1d362daacc630efc15fc30244f55d0.png)

### 请求的处理流程

-   

    ![desc](media/9ef593ea8f244a8dd943ba92bac3e98b.png)

-   

    ![desc](media/5a2ecdfad05193f14cc767bf9597aeea.png)

-   

    ![desc](media/42249ab340c717dbcab5624c6880cd34.png)

# handler 模块

## 

![desc](media/82f080a2835edec9b6cf0aae815320a4.png)

## 模块的基本结构

### 模块配置结构

-   

    ![desc](media/9dac0821717fed406b297b775befd6dc.png)

### 模块配置指令

-   

    ![desc](media/64841277a19ec896abcf3f3fcf981104.png)

-   ngx_command_t一个模块的配置指令是定义在一个静态数组中的

    •

    ![desc](media/50283056fff6ac80081936809dafb226.png)

    • type

    •

    ![desc](media/57719ca06bbf92a5f6ff92f55f80aadc.png)

    •

    ![desc](media/be04fe2c30747a094a0e514982d0e09e.png)

    •

    ![desc](media/1cb69d638ef544be9e91e909058bdcde.png)

    • set

    •

    ![desc](media/d8e30d21b34b52f4bbdae484a41a4673.png)

    •

    ![desc](media/39c63e447a860c583deafcbc40b83efc.png)

    • conf

    •

    ![desc](media/c0348c0e9334350ef3975862d0e93825.png)

    • offset

    •

    ![desc](media/edc0af661bba3e08afe5d50183364cb8.png)

    • post

    •

    ![desc](media/2772b90993e3d34876bb63c0b277b74e.png)

    • 需要注意的是，就是在ngx_http_hello_commands这个数组定义的最后，都要加一个ngx_null_command作为结尾。

-   模块上下文结构

    •

    ![desc](media/9f90b3661b39765732715ad69e9beb07.png)

    •

    ![desc](media/8377f5399343b4f9e3ab436276083753.png)

-   模块的定义

    •

    ![desc](media/49d2a4af649c739fa1d2c6f3a50ef5c8.png)

## handler 模块的基本结构

### 

![desc](media/d0345236a87e537d43283ff0923bb412.png)

## handler 模块的挂载

### 按处理阶段挂载

-   

    ![desc](media/6464bfb5844a0d42c636d4a9911e86a6.png)

-   

    ![desc](media/3dab50851a0d5d3c7e2461a7a291dae5.png)

### 按需挂载

-   

    ![desc](media/d6351c68abef480d53a3d9b9820e06bf.png)

-   

    ![desc](media/4cdff3e9e87889c9b6eda2d6fc5bcd1b.png)

### handler 的编写步骤

-   1、编写模块基本结构。包括模块的定义，模块上下文结构，模块的配置结构等。2、实现 handler 的挂载函数。根据模块的需求选择正确的挂载方式。3、编写 handler 处理函数。模块的功能主要通过这个函数来完成。

### Nginx 示例

-   

    ![desc](media/e75400173f6f508ad75e6533411c5139.png)

-   完整示例

    •

    ![desc](media/69a7021a7c2de332b50de888ff5aa6d1.png)

-   w3cschool.cn-Nginx 示例 hello handler 模块.pdf

### handler 模块的编译和使用

-   config 文件的编写

    •

    ![desc](media/ca2b47e8eeb4e558b9ed66465b7d5361.png)

-   编译

    •

    ![desc](media/732f4cb7493f57a8c04eb5037a3506df.png)

-   使用

    •

    ![desc](media/5c0463ff2bcc9cdfcd6566bb93014c42.png)

### 更多 handler 模块示例分析

-   http access module

    •

    ![desc](media/c88bd2611de1dd85ab8beb6d7043f426.png)

-   http static module

    •

    ![desc](media/121c42f21fd1030edc70c0036ed4f475.png)

    •

    ![desc](media/43147db465da87ae7c71c3b1149f48eb.png)

-   http log module

    •

    ![desc](media/2da4a2f2703ce5b12a62a4fb4696e71a.png)

# 过滤模块

## 简介

### 执行时间和内容

-   

    ![desc](media/e3bf8198f422ac6b083db3feb5ff1218.png)

### 执行顺序

-   

    ![desc](media/d1ef8aa0531afbec6de6b9da6bad919c.png)

-   

    ![desc](media/5756aed5858e683bbbe3b6fcfc181073.png)

### 模块编译

-   

    ![desc](media/0e6a068c546a62857c9e93801b4d8337.png)

## 过滤模块的分析

### 相关结构体

-   

    ![desc](media/2322dcb4be835787b009248d8e52d4f6.png)

-   

    ![desc](media/52a198930ef145c6485f67d69cd9f517.png)

### 响应头过滤函数

-   

    ![desc](media/35381b78e90a6857b63f8d6fd33f9d63.png)

### 响应体过滤函数

-   

    ![desc](media/d67f6b1e8a1f471279c1a8967c7bf14d.png)

### 主要功能介绍

-   

    ![desc](media/537762f7d098f031264f3f8710cfa40a.png)

### 发出子请求

-   

    ![desc](media/485f31201dc06b7c1e76ae95680f0559.png)

### 优化措施

-   

    ![desc](media/33550143be8e07859d113121221c0d36.png)

### 过滤内容的缓存

-   

    ![desc](media/d09b656b16ff1aa2397d4d78a8fe4c5c.png)

# upstream 模块

## 模块简介

### 

![desc](media/bbc3e8c75ec8ac95a330785da4fa07d3.png)

### 

![desc](media/5ce723e0e57a2f80dcf1697c5104cdf1.png)

## 模块接口

### 

![desc](media/ae905f22f9decd04b2f69f7446ff733f.png)

## memcached 模块分析

### 

![desc](media/8e1f35e51959c6d33a9b395d3507e2d8.png)

## handler 模块的接入方式

### 

![desc](media/51dba3e1eed730f924a4f6086a46a1b8.png)

## Upstream 模块

### 

![desc](media/371b8d24dc93653f6c5afa374e7c4bac.png)

## 回调函数

### 

![desc](media/d8a6e4155701f45b5f50a4048ace4c80.png)

# 负载均衡模块

## 

![desc](media/4d71481dc7ec03c69f19e9bd86351756.png)

## 

![desc](media/bf8482c41725aea87103d07c5308726c.png)

## 配置

### 

![desc](media/72e85649437a398d12d326a0235b4c6e.png)

## 指令

### 

![desc](media/b49ab4aa23187b147d92923eeee4b404.png)

## 钩子

### 

![desc](media/d01fa0229f5912eafa1f33533892329a.png)

## 设置 uscf-\>flags

### 

![desc](media/ef9c0a016c5bc787bb1f78b9bd501be3.png)

## 设置 init_upstream 回调

### 

![desc](media/7ade3be8875b5bc196c28d7da3654456.png)

## 初始化配置

### 

![desc](media/8c4f6fd51b9c5e30e0fef891ce1d7f06.png)

## 初始化请求

### 

![desc](media/32cc33c440c3c515defa44e6239acebf.png)

## peer.get 和 peer.free 回调函数

### 

![desc](media/74fe7ccab84a1a84ee4cc7aca3d00c23.png)

## upstream 整体流程

### 

![desc](media/b50fa1b7615d94ed5508286e415ed9ad.png)

# core 模块

## 

![desc](media/4bc5903bf3ce6db8006b0499b3901d5e.png)

# event 模块

## event 的类型和功能

### 

![desc](media/fed964ab8dc0b655b8783cf02a33f498.png)

## accept 锁

### 

![desc](media/4dc4269ec68b1cbeb3a45c5ce462a3c2.png)

## 定时器

### 

![desc](media/0bfb1c00576c205498c67d3d34f29036.png)

# w3cschool.cn-Nginx 配置文件nginxconf中文详解.pdf
