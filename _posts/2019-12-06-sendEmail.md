---
layout: post
title: sendEmail使用TLS发送邮件
date: 2019-11-26
categories: blog
tags: [sendEmail]
description: 文章金句。
---

# sendEmail使用TLS发送邮件

* 要下载支持TLS的版本`http://caspian.dotconf.net/menu/Software/SendEmail/sendEmail-v156.zip`


* 在使用sendEmail启用tls发送邮件的时候出现

```bash
*******************************************************************
 Using the default of SSL_verify_mode of SSL_VERIFY_NONE for client
 is deprecated! Please set SSL_verify_mode to SSL_VERIFY_PEER
 possibly with SSL_ca_file|SSL_ca_path for verification.
 If you really don't want to verify the certificate and keep the
 connection open to Man-In-The-Middle attacks please set
 SSL_verify_mode explicitly to SSL_VERIFY_NONE in your application.
*******************************************************************
  at ./sendEmail.pl line 1906.
invalid SSL_version specified at /usr/share/perl5/vendor_perl/IO/Socket/SSL.pm line 444.
```

* 参考资料`http://caspian.dotconf.net/menu/Software/SendEmail/`

* 我的CentOS 7，则安装

```bash
yum -y install perl-IO-Socket-SSL openssl-perl openssl-devel
```

* 不用降级perl也可以的方法

```
修改 sendEmail 脚本：

第 1906 行，将 'SSLv3 TLSv1' 修改为 'SSLv23:!SSLv2'：
```
