---
layout: post
title: uwsgi gunicorn nginx 配置
date: 2020-04-21
categories: uwsgi gunicorn 
tags: [uwsgi gunicorn]
description: 文章金句。
---

# uwsgi gunicorn nginx 配置

## 介绍
* WSGI（Web Server Gateway Interface），翻译为Python web服务器网关接口，即Python的Web应用程序（如Flask）和Web服务器(如Nginx)之间的一种通信协议。也就是说，如果让你的Web应用在任何服务器上运行，就必须遵循这个协议。

* 那么实现WSGI协议的web服务器有哪些呢？就比如uWSGI与gunicorn。两者都可以作为Web服务器。可能你在许多地方看到的都是采用Nginx + uWSGI（或gunicorn）的部署方式。实际上，直接通过uWSGI或gunicorn直接部署也是可以让外网访问的，那你可能会说，那要Nginx何用？别急，那么接来下介绍另一个Web服务器——Nginx

* Nginx作为一个高性能Web服务器，具有负载均衡、拦截静态请求、高并发...等等许多功能，你可能要问了，这些功能和使用Nginx + WSGI容器的部署方式有什么关系？
* 首先是负载均衡，如果你了解过OSI模型的话，其实负载均衡器就是该模型中4~7层交换机中的一种，它的作用是能够仅通过一个前端唯一的URL访问分发到后台的多个服务器，这对于并发量非常大的企业级Web站点非常有效。在实际应用中我们通常会让Nginx监听（绑定）80端口，通过多域名或者多个location分发到不同的后端应用。

* 其次是拦截静态请求，简单来说，Nginx会拦截到静态请求（静态文件，如图片），并交给自己处理。而动态请求内容将会通过WSGI容器交给Web应用处理;

* Nginx还有其他很多的功能，这里便不一一介绍。那么前面说了，直接通过uWSGI或gunicorn也可以让外网访问到的，但是鉴于Nginx具有高性能、高并发、静态文件缓存、及以上两点、甚至还可以做到限流与访问控制，所以选择Nginx是很有必要的；

* 这里可以说明，如果你选择的架构是：Nginx + WSGI容器 + web应用，WSGI容器相当于一个中间件；如果选择的架构是uWSGI + web应用，WSGI容器则为一个web服务器



## uwsgi配置
* [官方文档](https://uwsgi-docs.readthedocs.io/en/latest/)

p.ini
```bash
[uwsgi]
#socket=127.0.0.1:5003
# uwsgi的监听端口                                              

socket=/tmp/p.sock
chmod-socket=600

buffer-size=65536                                                                              
#设置用于uwsgi包解析的内部缓存区大小，默认是4k                                                                               
chdir=/opt/p_test 
# 项目根目录                                           
                                                                               
wsgi-file=p.py       
# flask程序的启动文件                                           
                                                                     
callable=app
#flask应用实例的名称，是flask独有的配置项       

# process-related settings
# master
master=true
# maximum number of worker processes
processes=4
# the socket (use the full path to be safe
enable-threads=true
threads=4
thunder-lock=true

pidfile=uwsgi.pid

vacuum = true
#退出后清理环境

#daemonize=/tmp/vancouver.log 
#使进程在后台运行,并将日志打到指定的日志文件

uid=99
gid=99
#指定运行用户

#服务统计
stats=127.0.0.1:5002
#stats-server=true
stats-http=true

static-map=/static=/opt/p_test
#uWSGI 即可将指定请求前缀映射到文件系统上的对应物理目录
```
* 启动方式

```
uwsgi --ini p.ini

```


## gunicorn
* [官方文档](http://docs.gunicorn.org/en/latest)
* 安装依赖

```
pip3 install gevent
```

gconf.py
```python
from gevent import monkey
monkey.patch_all()
import multiprocessing
debug = True
loglevel = 'debug'
bind = '0.0.0.0:5000' #绑定与Nginx通信的端口
pidfile = 'log/gunicorn.pid'
logfile = 'log/debug.log'
accesslog='log/debug1.log'
#loglevel='info'
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = 'gevent' #默认为阻塞模式，最好选择gevent模式
worker_connections=10240 #
backlog=5000 #挂起连接的最大数目
max_requests_jitter=3 #抖动参数，防止worker全部同时重启

```

* 启动方式

```
gunicorn -c gconf.py g:app -D
```



## nginx配置

```bash
server {                                                                       
	listen 8000;                   # 服务器监听端口                                                 
	#server_name test.com; # 这里写你的域名或者公网IP                                                    
	charset      utf-8;          # 编码                                                  
	client_max_body_size 75M;                                                       

 location / {                                                                   
	include uwsgi_params;         # 导入uwsgi配置                                            
	#uwsgi_pass 127.0.0.1:5002;    # 转发端口，需要和uwsgi配置当中的监听端口一致                                             
	uwsgi_pass unix:///tmp/p.sock;    # 转发sock，需要和uwsgi配置当中的监听一致                                             
	# uwsgi_param UWSGI_PYTHON /home/自己创建的目录/venv;       # Python解释器所在的路径（这里为虚拟环境）      
	# uwsgi_param UWSGI_CHDIR /home/自己创建的目录;             # 项目根目录     
	# uwsgi_param UWSGI_SCRIPT manage:app; #比如你测试用test.py文件，文件中app = Flask(__name__)，那么这里就填  test：app
			# 项目的主程序                           
      }  
 location /static {
  alias /opt/p_test/static;
  #静态文件不走uwsgi直接nginx访问
 }
	  
}
```