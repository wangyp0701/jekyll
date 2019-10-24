---
layout: post
title: multivisor文档
date: 2019-8-28
categories: blog
tags: [multivisor]
description: 文章金句。
---
# multivisor

## 配置server

* 下载wget https://github.com/tiagocoutinho/multivisor/archive/5.0.2.tar.gz

* wget http://mirrors.yappam.com/opt/linux/node-v10.16.2-linux-x64.tar.xz

* 安装 nodejs pip等。

* 依赖 python-devel python-setuptools gcc-c++

* pip install --upgrade setuptools

  ```bash
  ＃获取项目：
  git clone https://github.com/tiagocoutinho/multivisor
  cd multivisor
  
  
  ＃安装前端依赖项
  npm install
  ＃通过缩小生成生产
  npm run build
  
  ＃安装后端：随意使用自己喜欢的Python的虚拟环境
  
  pip install .
  
  ＃启动一些例子supervisor
  mkdir examples/full_example/log
  supervisord -c examples/full_example/supervisord_lid001.conf
  supervisord -c examples/full_example/supervisord_lid002.conf
  supervisord -c examples/full_example/supervisord_baslid001.conf
  
  ＃最后，启动multivisor： 
  multivisor -c examples/full_example/multivisor.conf
  ```


## 配置其他主机

* pip install --upgrade setuptools

* pip install supervisor multivisor

* 通过将以下行添加到supervisord.conf来配置multivisor rpc接口：

  ```
  [rpcinterface:multivisor]
  supervisor.rpcinterface_factory = multivisor.rpc:make_rpc_interface
  bind=*:9002
  ```

  如果没有给出*绑定*，则默认为`*:9002`。

  对每个运行的主管重复上述步骤。
  
* supervisord启动文件

```bash
[Unit]
Description=Process Monitoring and Control Daemon
After=rc-local.service nss-user-lookup.target

[Service]
Type=forking
ExecStart=/usr/bin/supervisord -c /etc/supervisord.conf

[Install]
WantedBy=multi-user.target

```

