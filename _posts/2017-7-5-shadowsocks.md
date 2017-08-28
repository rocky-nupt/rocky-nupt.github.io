---
layout: post
title: centos命令行使用shadowsocks代理
---

文章以centos为例，其他linux系统类似。文末还有docker设置http代理。

系统：centos 7 minimal

### 安装客户端shadowsocks

    yum install epel-release
    yum install python-pip
    pip install shadowsocks
新建配置文件

    vim /etc/shadowsocks.json
    {
        "server":"your_server_ip",      #ss服务器IP
        "server_port":your_server_port, #端口
        "local_address": "127.0.0.1",   #本地ip
        "local_port":1080,              #本地端口
        "password":"your_server_passwd",#连接ss密码
        "timeout":300,                  #等待超时
        "method":"rc4-md5",             #加密方式
        "fast_open": false,             # true 或 false。如果你的服务器 Linux 内核在3.7+，可以开启 fast_open 以降低延迟。
        "workers": 1                    # 工作线程数
    }
    
用配置文件建立连接

    nohup sslocal -c /etc/shadowsocks.json /dev/null 2>&1 &

### 安装转发代理Privoxy

    yum install privoxy

查看vim /etc/privoxy/config文件，先搜索关键字:listen-address找到listen-address  127.0.0.1:8118这一句，保证这一句没有注释，8118就是将来http代理要输入的端口。然后搜索forward-socks5t,将forward-socks5t / 127.0.0.1:1080 .此句的注释去掉. 

启动privoxy

    privoxy /etc/privoxy/config
配置环境变量

    vim /etc/profile
    export http_proxy=http://127.0.0.1:8118
    export https_proxy=http://127.0.0.1:8118
######
    source /etc/profile

验证:

    curl google.com

### docker设置http代理

    mkdir -p /etc/systemd/system/docker.service.d
    vim /etc/systemd/system/docker.service.d/http-proxy.conf

    [Service]
    Environment="HTTP_PROXY=http://[proxy-addr]:[proxy-port]/" "HTTPS_PROXY=https://[proxy-addr]:[proxy-port]/"
       
如果还有内部的不需要使用代理来访问的Docker registries，那么嗨需要制定NO_PROXY环境变量：

    [Service]
    Environment="HTTP_PROXY=http://[proxy-addr]:[proxy-port]/" "HTTPS_PROXY=https://[proxy-addr]:[proxy-port]/" "NO_PROXY=localhost,127.0.0.1,docker-registry.somecorporation.com"  

重启docker

    systemctl daemon-reload
    systemctl restart docker

