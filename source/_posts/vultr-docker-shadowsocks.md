---
title: vultr+docker+shadowsocks
date: 2018-11-10 14:53:26
tags:
categories: VPN
---

## vultr
到 [vultr官网](https://www.vultr.com)购买服务器  
选择5$的那个基本上就够用了，在application中选择docker  

## shadowsocks
#### 拉取docker-shadowsocks镜像  
`docker pull oddrationale/docker-shadowsocks`
#### 配置shadowsocks服务
`docker run -d -p 1996:1996 oddrationale/docker-shadowsocks -s 0.0.0.0 -p 1996 -k yourpasswd -m aes-256-cfb`
#### 检查是否成功
`docker ps`