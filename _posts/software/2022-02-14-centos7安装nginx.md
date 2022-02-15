---
layout: post
title: centos7安装nginx
category: software
tags: [software]
keywords: 安装软件
---
centos7 安装nginx步骤(本文默认进入是/root)

## 1.下载nginx:[下载地址](http://nginx.org/en/download.html)
或者直接使用命令下载 
```
wget http://nginx.org/download/nginx-1.20.1.tar.gz  
```
## 2.解压
```
tar -zvxf nginx-1.20.1.tar.gz  
```
## 3.依赖下载
```
yum -y install gcc gcc-c++
yum -y install zlib-devel
```
## 4.预编译
进入解压后的目录  /nginx-1.20.1进行预编译 
```
./configure --prefix=/usr/local/nginx  ps:--prefix是指定安装目录(建议指定,本人是安装在./usr/local/nginx下)
make && make install
```
## 5.启动nginx
```
cd /usr/local/nginx/sbin
./nginx
```
然后浏览器输入http://ip:80
如果出现访问不了,可以尝试暂时关闭防火墙,systemctl stop firewalld.service (阿里云等记得配置规则组放开80端口)，然后再刷新浏览器地址。
永久关闭防火墙 systemctl disable firewalld.service  



    
