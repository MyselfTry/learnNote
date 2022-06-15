**Nginx使用**

# Nginx概述

## HTTP基础功能

### 

![desc](media/8a803ef488cd0308edb046f50162d61b.png)

## IMAP/POP3 代理服务功能

### 

![desc](media/47636444ac3d3ac7cf93edf1db160b47.png)

## 支持的操作系统

### 

![desc](media/6475d11e1aaab025b9a379f5b4f7b01f.png)

## 结构与扩展

### 

![desc](media/768bb7868afd20a1d03d63334b0d4dac.png)

## 其他HTTP功能

### 

![desc](media/a97022124a8722a9f02361b44fa4ecc1.png)

## 实验特性

### 

![desc](media/ae30c78a0855bae291fa80303217083f.png)

## 优越性

### 

![desc](media/f199c3f539fc91a62f45085c18c7592a.png)

# nginx配置

## 安装Nginx

### 

![desc](media/96710327e18a7d4c78dbe2169e6cd310.png)

## 命令行参数和信号

### nginx命令行参数

![desc](media/93d22748b44e586fd80ef67f6772c0d7.png)

### nginx 控制信号

![desc](media/3b5dacb2e1576f53125aade7c3e5851b.png)

### nginx 启动、停止、重启命令

-   

    ![desc](media/2f023f8650b361a1bf53ea8cebf5d019.png)

-   

    ![desc](media/d23ba3d65271e7beddef8d4e9c889fd1.png)

    •

    ![desc](media/e686be95484be22bea04492918589409.png)

    •

    ![desc](media/dff50dcd944d0dcbdf14132bd50c5add.png)

### 优化 Nginx

-   

    ![desc](media/0ceaa7fa42a503251abdc2be90d8b30d.png)

-   

    ![desc](media/a0098c600740d94b66fd963c8b9901f6.png)

## rewrite配置

### Nginx正则匹配符说明

![desc](media/49a056db2c9c9f8d4464fda36eb3555e.png)

### 全局变量

![desc](media/cbc4a29ce644abff1754bb7fe838e83d.png)

-   

    ![desc](media/8c40dfaa813faa91f9917feae0b5b99b.png)

### rewrite配置实例

-   重定向规则

    ![desc](media/d1ab38e6571042e6433b844ebc8f31a7.png)

-   多目录转成参数

    ![desc](media/734250616bcd6d725dc2d6c11a4f58a2.png)

-   目录对换

    ![desc](media/53bd2be78154ddbdec6731cb0819fe61.png)

-   目录自动加“/” ，这个功能一般浏览器自动完成

    ![desc](media/4122b8be2bd432588066008bf096cda4.png)

-   禁止htaccess

    ![desc](media/89ed61bb49acedfbde4b1f59acbb3c59.png)

-   禁止多个目录

    ![desc](media/4717745ca5ff135fff3fcf5fcd3b1d96.png)

-   禁止以/data开头的文件，可以禁止/data/下多级目录下.log.txt等请求

    ![desc](media/ca2a6a57b268910e5430f774a8fdf667.png)

-   文件的保质期时间

    ![desc](media/a325dc3c975075e91f68af2ec3e25cb0.png)

-   防盗链的设置

    ![desc](media/9fa607d7be9751e14b0c575edde6db65.png)

    • 文件反盗链并设置过期时间

    ![desc](media/c0f1d4bb2077fcf64e4c3768678dc59d.png)

-   只充许固定ip访问网站，并加上密码；这个对有权限认证的应用比较在行

    ![desc](media/57bda81ff4f8fc7fa0f3fe0947265290.png)

-   文件和目录不存在的时重定向

    ![desc](media/8bd0e4a8d073eaf758cf1f477bc93558.png)

-   域名跳转

    ![desc](media/eb854802dfb5de0b15f61d4bff5c44a8.png)

-   多域名转向

    ![desc](media/cf582a23b580c08672df1595d1096a02.png)

-   三级域名跳转

    ![desc](media/45fab25e268f4e7c2effe22a48759ae5.png)

-   域名镜向

    ![desc](media/38550c0d79fcbb6137d159df52722b27.png)

-   某个子目录作镜向,这里的示例是demo子目录

    ![desc](media/d2767f39f3353b9a8f74a175933ced9c.png)

# 核心模块

## 主模块

### 

![desc](media/2f7311d6438189e67d00a5ff46dd054d.png)

### 

![desc](media/cea0017ff7139bbe8e15004bb4ab6d9a.png)

### 

![desc](media/dc96390dab95ab07c46ddfc65833f85f.png)

### 

![desc](media/26bc64b59616a67b7bd5a263b541cd31.png)

### 

![desc](media/123fff1ee559702dc4bf559ddc90aea1.png)

### 

![desc](media/f774fe7b1254f9c259f3ed00ace63a1e.png)

### 

![desc](media/3b2c2ddd33bd56d4de097a855a7fabb9.png)

### 

![desc](media/6c017b27886cb50165c0d0a5a4607bcd.png)

### 

![desc](media/17ef512d87e2c77f36f21a1a09d189f4.png)

### 

![desc](media/d2ef5f4d60104588826d5dd5adf9b151.png)

### 

![desc](media/3773ede4ad5813918891b0e564b669ff.png)

### 

![desc](media/463ba82c7159cc7ea2f86ab88e9c6387.png)

### 

![desc](media/826cbf97306d57e6af8390d065f6654e.png)

## 事件模块

## HTTP模块

# HTTP模块

## Access 模块

### 

![desc](media/7184527ec5f8699f64e121b129facf3f.png)

### 

![desc](media/21e0ab8a8df4bf06e5bf867c18596614.png)

## Auth Basic 模块

### 

![desc](media/463c4af51224b0a4b70355f046e22a4f.png)

### 

![desc](media/b2a1b8d320dc315fdf6ea1c01217f1b0.png)

## AutoIndex 模块

### 

![desc](media/fa28727d465ff9c42739008b289fa986.png)

### 

![desc](media/f69b76f491f6483104452520b128a3f9.png)

## DAV模块

### 

![desc](media/a0d4485d6b6683c28892a5324ca61c6b.png)

### 

![desc](media/bf84dbc1528a80a28c8e3ea53a569235.png)

## FastCGI 模块

### 

![desc](media/df54e602229d461692784d2d39ab837b.png)

### 

![desc](media/3ef404298e5e205002139c37ea56229e.png)

### 

![desc](media/b1e3909fc4ed729410840c68a35ebf02.png)

# 其他模块

## 限制并发

### 

![desc](media/eeb6346a6e2636226ba6e5b3b5b377e8.png)

## 限制IP访问频率

### 

![desc](media/dcb5006ff3c3d9cbc52fc97071f0383a.png)

## 限制IP带宽占用

### 

![desc](media/80fb36ccada37b490ffcecafd5041829.png)

## 配置SSL

### 

![desc](media/c21c2654b623ccff2b6f91a55aca476e.png)

## 静态资源缓存

### 

![desc](media/4896f7aeb95d3a6aeda16f5ca7800434.png)
