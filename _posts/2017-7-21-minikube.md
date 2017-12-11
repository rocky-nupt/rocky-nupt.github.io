---
layout: post
title: minikube安装
---

minikube安装教程

环境：centos7<br />
虚拟化方式：kvm

### 安装kvm

检测（有输出即可）：
 
    egrep '(vmx|svm)' /proc/cpuinfo
安装

    yum install qemu-kvm qemu-img virt-manager libvirt libvirt-python python-virtinst libvirt-client virt-install

查看kvm安装情况：lsmod | grep kvm<br />
类似下面的输出:

    kvm_intel             200704  0
    kvm                   593920  1 kvm_intel

### 翻墙
见翻墙篇：<a href="http://rocky-nupt.github.io/shadowsocks/">http://rocky-nupt.github.io/shadowsocks/</a>

### 安装kubectl
    curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl 
    chmod +x kubectl<br />
    mv kubectl /usr/local/bin

### 下载安装minikube

先下docker-machine-driver-kvm插件：

    curl -L https://github.com/dhiltgen/docker-machine-kvm/releases/download/v0.7.0/docker-machine-driver-kvm -o /usr/local/bin/docker-machine-driver-kvm    
    chmod +x /usr/local/bin/docker-machine-driver-kvm
    yum install libvirt-daemon-kvm kvm

再下minikube 

    curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.21.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

### 启动

    minikube config set vm-driver kvm
    minikube start

启动失败的话可以重启一下

### minikube中docker翻墙
见翻墙篇：<a href="http://rocky-nupt.github.io/shadowsocks/">http://rocky-nupt.github.io/shadowsocks/</a>
