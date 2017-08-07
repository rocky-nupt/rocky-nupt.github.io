---
layout: post
title: devstack安装
---

devstack的详细安装过程。

环境：ubuntu 16.04 server或者desktop都可以<br />
内存建议8G，硬盘建议50G<br />
建议设置root密码

切换到root用户：

    su -
    cd /home

获取devstack：

    git clone https://github.com/openstack-dev/devstack.git

创建stack用户：

    cd /home/devstack/tools
    ./create-stack-user.sh

给stack赋予权限：

    chown -R stack:stack /home/devstack

设置本地pip源加速安装：
    mkdir /opt/stack/.pip
    mkdir /root/.pip
分别在两个文件夹中创建相同的文件pip.conf, 并写下下面的内容：
    [global]
    trusted-host=mirrors.aliyun.com
    index-url = http://mirrors.aliyun.com/pypi/simple/

准备/home/devstack/local.conf文件：

    cd /home/devstack
    vim local.conf
写入以下内容：

    [[local|localrc]]
    SERVICE_TOKEN=123456
    ADMIN_PASSWORD=123456
    MYSQL_PASSWORD=123456
    RABBIT_PASSWORD=123456
    SERVICE_PASSWORD=$ADMIN_PASSWORD
    
    # 这里是使用的国内的 openstack 源，但是偶尔会出现一些滞后，当网络较好时建议不使用。
    GIT_BASE=http://git.trystack.cn
    NOVNC_REPO=http://git.trystack.cn/kanaka/noVNC.git
    SPICE_REPO=http://git.trystack.cn/git/spice/spice-html5.git
    
    disable_service n-net
    enable_service q-svc
    enable_service q-agt
    enable_service q-dhcp
    enable_service q-l3
    enable_service q-meta
    enable_service neutron
    enable_service s-proxy s-object s-container s-account
    
    # 如果环境中已经有了一些安装包和 openstack 源码，可以设置这两项为 Yes 和 True，以保
    证不出现问题。
    RECLONE=yes
    PIP_UPGRADE=True
    HOST_IP=127.0.0.1
    
    # 默认安装路径在 stack 用户目录， /opt/stack/中
    DEST=/opt/stack/
    LOGDIR=$DEST/logs
    LOG_COLOR=False
    LOGFILE=$DEST/logs/stack.sh.log
    LOGDAYS=2
    SWIFT_HASH=66a3d6b56c1f479c8b4e70ab5c2000f5
    
    # devstack 支持的服务需要 enable_service，还有部分服务是以 devstack 插件的形式存在, 例
    # 如：
    # enable_plugin sahara https://git.openstack.org/openstack/sahara
    # enable_plugin heat https://git.openstack.org/openstack/heat
    # enable_service ceilometer
    enable_service tempest

执行安装：
    su stack
    /home/devstack/stack.sh

等待安装结束，中途断掉只要重新运行最后一步的命令就行，安装结束后可以登陆http://127.0.0.1，账号admin，密码123456