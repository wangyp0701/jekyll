---
layout: post
title: 'Nginx 安装 GEOIP实现屏蔽指定城市访问'
subtitle: 'nginx'
date: 2019-11-04
categories: 技术
cover: ''
tags: nginx geoip
---


# Nginx 安装 GEOIP实现屏蔽指定城市访问

* Nginx可以通过设置IP段来屏蔽某些IP访问，但是针对某些程序，比如我们要设置禁止北京访问，这个需求就比较难以实现，首先你需要收集全部北京的IP段，而这一步可能就困死大部分人了。这里推荐用GEOIP模块，安装上后只需要简单设置就能实现屏蔽指定地区城市访问你的网站



* 首先确认已安装的 Nginx 是否安装并启用了 GeoIP 模块
```bash
# nginx -V
nginx version: nginx/1.12.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-11) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --user=nginx --group=nginx --with-http_geoip_module --with-http_ssl_module --with-http_realip_module --with-http_addition_module --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_random_index_module --with-http_secure_link_module --with-http_stub_status_module --with-mail --with-mail_ssl_module --with-file-aio
```
* 如果看到其中有 –with-http_geoip_module，则说明已经支持该模块，如果没有就是不支持.那就继续往下，我们安装 GeoIP。



* 首先安装依赖


```bash
# yum -y install zlib zlib-devel
# wget http://geolite.maxmind.com/download/geoip/api/c/GeoIP.tar.gz
# tar -zxvf GeoIP.tar.gz
# cd GeoIP-1.4.8
# ./configure
# make
# make install
```
* 使用ldconfig将库索引到系统中
```bash
# echo '/usr/local/lib' > /etc/ld.so.conf.d/geoip.conf
# ldconfig
```
* 检查库是否加载成功
```bash
# ldconfig -v | grep GeoIP
libGeoIPUpdate.so.0 -> libGeoIPUpdate.so.0.0.0
libGeoIP.so.1 -> libGeoIP.so.1.4.8
libGeoIPUpdate.so.0 -> libGeoIPUpdate.so.0.0.0
libGeoIP.so.1 -> libGeoIP.so.1.5.0
```
* 将 GeoIP 模块编译到 Nginx 中
* 根据你当前 Nginx 的安装参数带上 –with-http_geoip_module 重新编译
```bash
# ./configure --user=nginx --group=nginx \
    --with-http_geoip_module \
    --with-http_ssl_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_random_index_module \
    --with-http_secure_link_module \
    --with-http_stub_status_module \
    --with-mail \
    --with-mail_ssl_module \
    --with-file-aio
# make && make install
```
* 如果系统是全新的，那就参考下面安装方法


```bash
# wget https://nginx.org/download/nginx-1.12.2.tar.gz
# tar zxvf nginx-1.12.2.tar.gz
# cd nginx-1.12.2
# ./configure --user=nginx --group=nginx \
    --with-http_geoip_module \
    --with-http_ssl_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_random_index_module \
    --with-http_secure_link_module \
    --with-http_stub_status_module \
    --with-mail \
    --with-mail_ssl_module \
    --with-file-aio
# make && make install
```
* GeoIP使用方法
* 首先查看本地是否已有 GeoIP 数据库
```bash
# cd /usr/local/share/GeoIP
# ll
-rw-r--r--. 1 root root  1183408 Mar 31 06:00 GeoIP.dat
-rw-r--r--. 1 root root 20539238 Mar 27 05:05 GeoLiteCity.dat
```

* 如果没有这两个库，则手动下载
```bash
#wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz
#gzip GeoLiteCity.dat.gz
#wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCountry/GeoIP.dat.gz
#gzip GeoIP.dat.gz
```
* 将库地址配置到 nginx.conf 中这个位置
```bash
http{
    ...
    geoip_country /usr/local/share/GeoIP/GeoIP.dat;
    geoip_city /usr/local/share/GeoIP/GeoLiteCity.dat;
    server {
        location / {
            root /www;
            if( $geo_country_code = CN ){
                root /www/zh;
            }
        }
    }
}
```
* 这样，当来自中国的 IP 访问网站后就自动访问到预定的 root /www/zh 下的页面

* 其他参数

```bash
$geoip_country_code; – 两个字母的国家代码，如：”RU”, “US”。

$geoip_country_code3; – 三个字母的国家代码，如：”RUS”, “USA”。

$geoip_country_name; – 国家的完整名称，如：”Russian Federation”, “United States”。

$geoip_region – 地区的名称（类似于省，地区，州，行政区，联邦土地等），如：”30”。 30代码就是广州的意思

$geoip_city – 城市名称，如”Guangzhou”, “ShangHai”（如果可用）。

$geoip_postal_code – 邮政编码。

$geoip_city_continent_code。

$geoip_latitude – 所在维度。

$geoip_longitude – 所在经度。
```

```bash
CN,01,”Anhui”
CN,02,”Zhejiang”
CN,03,”Jiangxi”
CN,04,”Jiangsu”
CN,05,”Jilin”
CN,06,”Qinghai”
CN,07,”Fujian”
CN,08,”Heilongjiang”
CN,09,”Henan”
CN,10,”Hebei”
CN,11,”Hunan”
CN,12,”Hubei”
CN,13,”Xinjiang”
CN,14,”Xizang”
CN,15,”Gansu”
CN,16,”Guangxi”
CN,18,”Guizhou”
CN,19,”Liaoning”
CN,20,”Nei Mongol”
CN,21,”Ningxia”
CN,22,”Beijing”
CN,23,”Shanghai”
CN,24,”Shanxi”
CN,25,”Shandong”
CN,26,”Shaanxi”
CN,28,”Tianjin”
CN,29,”Yunnan”
CN,30,”Guangdong”
CN,31,”Hainan”
CN,32,”Sichuan”
CN,33,”Chongqing”
```
