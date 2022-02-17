---
layout: post
title: centos7安装nacos
category: software
tags: [software]
keywords: 安装软件,centos7,nginx,安装nginx,centos7安装nginx
---
centos7 安装nacos步骤

## 1.下载nacos
###1.1windows下下载gz包[下载地址](https://github.com/alibaba/nacos/releases/download/1.4.3/nacos-server-1.4.3.tar.gz)
上传下载文件到/usr/local下

###1.2或者在centos下直接下载
```
cd /usr/local/
wget https://github.com/alibaba/nacos/releases/download/1.4.3/nacos-server-1.4.3.tar.gz
```

## 2.解压
```
tar -zvxf nacos-server-1.4.3.tar.gz  
cd /usr/local/nacos
```

##3.处理(前置条件是已安装mysql数据库)
  创建nacos的mysql数据库 
    1.进入到nacos后,复制其conf文件下的nacos-mysql.sql,在mysql中新建nacos数据库，运行此sql
    2.编辑conf文件夹下的application.properties文件
  ```
  vim application.properties
  ```
修改图中的这些配置
![](http://image.xiaoyaowind.com/image/1645013432.png)
然后保存退出
3.修改启动sh为单机启动
```
cd ../bin
vim startup.sh
```
修改
```
export MODE="standalone"
```
![](http://image.xiaoyaowind.com/image/1645013675(1).png)
保存退出
然后访问http//ip:8848/nacos/index.html
用户名和密码默认都是nacos,如果需要修改用户名请数据库修改，如果需要修改密码，请建立java web服务,maven中配置依赖包
```
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
代码处理示例
```
package com.xiaoyaowind.nacos.;

import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

/**
* Password encoder tool
*
* @author cat
  */
  public class PasswordEncoderUtil {

  public static void main(String[] args) {
        System.out.println(new BCryptPasswordEncoder().encode("cat"));
    }
}
```
然后把打印出的加密密码复制到数据库user表密码列中，搞定
