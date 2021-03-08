---
layout: post
title: openvpn 
date: 2021-03-08
categories: blog
tags: [openvpn,google_authenticator]
---

# 如何配置OpenVPN使用Google_Authenticator多因素认证

* 双因素认证
 双因素身份认证就是通过你所知道再加上你所能拥有的这二个要素组合到一起才能发挥作用的身份认证系统。双因素认证是一种采用时间同步技术的系统，采用了基于时间、事件和密钥三变量而产生的一次性密码来代替传统的静态密码。每个动态密码卡都有一个唯一的密钥，该密钥同时存放在服务器端，每次认证时动态密码卡与服务器分别根据同样的密钥，同样的随机参数（时间、事件）和同样的算法计算了认证的动态密码，从而确保密码的一致性，从而实现了用户的认证。说白了，就像我们几年前去银行办卡送的口令牌，以及网易游戏中的将军令，在你使用网银或登陆游戏时会再让你输入动态口令的。

* 产品种类
 市面上有基于硬件的，也有基于软件的产品，具体可以另搜啊，本人喜欢开源的东东，并找到了Google开源的二次认证系统[Google Authenticator](https://github.com/google/google-authenticator) ，可以利用智能手机生产30秒动态口令配合登陆，该验证器提供了一个六位数的一次性密码。目前ios 和Android 都有客户端供于下载。

* 功能
1. 实现登陆时，先输入用户帐号密码，在下一步输入动态口令。如果口令失败，不会进行登录。
2. 实现连接OpenVPN服务器时，结合客户端证书，帐号密码，通过动态验证码认证登录。
3. 部署完成后，即使手机客户端不能上网，整个二步验证系统还是可以正常运行的。
4. 动态口令在验证时用到了时间，所以要保持时间上的一致性。


## 安装openvpn google_authenticator_libpam

* 我这里以centos7为例安装
```bash
yum install openvpn easy-rsa google-authenticator -y
```

* 安装完成后将easy-rsa复制到/etc/openvpn

## 生成openvpn所需证书
* 编辑配置文件
```bash
cp -a /usr/share/easy-rsa/3.0.8 /etc/openvpn/easy-rsa
cd /etc/openvpn/easy-rsa

vim /etc/openvpn/easy-rsa/vars
set_var  EASYRSA_REQ_COUNTRY    "CN"
set_var  EASYRSA_REQ_PROVINCE    "CN"
set_var  EASYRSA_REQ_CITY    "Example City"
set_var  EASYRSA_REQ_ORG    "Example.org"
set_var  EASYRSA_REQ_EMAIL    "root@example.org"
set_var  EASYRSA_REQ_OU        "Example IT"
set_var  EASYRSA_KEY_SIZE    2048
set_var  EASYRSA_ALGO        rsa
set_var  EASYRSA_CA_EXPIRE    3650
set_var  EASYRSA_CERT_EXPIRE    3650
set_var  EASYRSA_CRL_DAYS    365
set_var  EASYRSA_DIGEST        "sha256"
```


* 启动PKI目录，并使用下面的命令建立CA密钥
./easyrsa init-pki
./easyrsa build-ca
* 并使用我们的CA证书签署“ vpn.example.org”密钥
./easyrsa build-server-full  vpn.example.org nopass
* 使用以下命令生成Diffie-Hellman密钥
./easyrsa gen-dh
* CRL（证书吊销列表）密钥将用于吊销客户端密钥
./easyrsa gen-crl



## pam  配置google_authenticator
* /etc/pam.d/下创建认证配置文件
```bash
vim /etc/pam.d/openvpn
auth    required     /usr/lib64/security/pam_google_authenticator.so secret=/home/${USER}/.google_authenticator authtok_prompt=pin
account required     pam_permit.so
```



## 生成客户端脚本

```bash
vim c-client.sh
# set the variables we'll use later
NAME_CLIENT="client0001"
MFA_LABEL="OPENVPN-${NAME_CLIENT}"
DIR_CLIENT="/etc/openvpn/clients/${NAME_CLIENT}"
 
# create the certificate and key
cd "/etc/openvpn"
/etc/openvpn/easy-rsa/easyrsa build-client-full "${NAME_CLIENT}" nopass
 
# create a directory to save all the files
mkdir -p "${DIR_CLIENT}"
 
# copy certificate, key, tls auth and CA
cp -v "/etc/openvpn/pki/ca.crt" "$DIR_CLIENT/vpn.example.org.ca.crt"
cp -v "/etc/openvpn/pki/ta.key" "$DIR_CLIENT/vpn.example.org.ta.key"
cp -v "/etc/openvpn/pki/issued/${NAME_CLIENT}.crt" "$DIR_CLIENT/"
cp -v "/etc/openvpn/pki/private/${NAME_CLIENT}.key" "$DIR_CLIENT/"
 
# copy and customize the client configuration
cp -v "/etc/openvpn/template_client.conf" "${DIR_CLIENT}/${NAME_CLIENT}.ovpn"
sed -i "s#CLIENT_NAME#${NAME_CLIENT}#g" "${DIR_CLIENT}/${NAME_CLIENT}.ovpn"
sed -i "s#PLATFORM_NAME#vpn.example.org#g" "${DIR_CLIENT}/${NAME_CLIENT}.ovpn"
 
# create a new local user
PASS=$(head -n 4096 /dev/urandom | tr -dc a-zA-Z0-9 | cut -b 1-10)
useradd -m "${NAME_CLIENT}"
echo "$PASS" | passwd --stdin ${NAME_CLIENT}
echo "$PASS" > ${DIR_CLIENT}/sshpass.txt
 
# run the google authenticator as the local user and save the code
su ${NAME_CLIENT} -c "/usr/local/bin/google-authenticator -C -t -f -D -r 3 -Q UTF8 -R 30 -w 3 -l ${MFA_LABEL} " > ${DIR_CLIENT}/authenticator_code.txt
```



## openvpn服务端配置

```bash
vim server.conf
plugin "/usr/lib64/openvpn/plugins/openvpn-plugin-auth-pam.so" "openvpn login USERNAME password PASSWORD pin OTP"

# tcp
port 11940
proto tcp
dev tun
persist-key
persist-tun
keepalive 10 900
comp-lzo
reneg-sec 0
 
# log
verb 3
mute 10
log-append /var/log/openvpn.log
status openvpn-status.log
  
# crypto
cipher AES-256-CFB8
auth SHA512
tls-auth pki/ta.key 0
tls-cipher TLS-DHE-RSA-WITH-AES-256-CBC-SHA
 
# certs
ca pki/ca.crt
cert pki/issued/vpn.example.org.crt
key pki/private/vpn.example.org.key
dh pki/dh.pem
crl-verify pki/crl.pem
 
# networking
server 10.1.1.0 255.255.255.0
push "route 192.168.0.0 255.255.255.0"
push "route 192.168.6.0 255.255.255.0"
push "dhcp-option DNS 192.168.0.234"

# clients
ifconfig-pool-persist ipp.txt
#client-config-dir ccd
max-clients 100
```

## client配置

```bash
vim client.ovpn
client
# vpn concentrator
remote vpn.yappam.com 11940
# certificates, with path relative to the config file
ca vpn.example.org.ca.crt
cert client0006.crt
key client0006.key
auth-user-pass
auth-nocache
# generic stuff
dev tun
proto tcp
nobind
persist-key
persist-tun
nobind
comp-lzo
verb 3
mute 10
reneg-sec 0 
# crypto
cipher AES-256-CFB8
tls-auth vpn.example.org.ta.key 1
tls-cipher TLS-DHE-RSA-WITH-AES-256-CBC-SHA
remote-cert-tls server
auth SHA512
script-security 2
static-challenge "Enter Google Authenticator Token" 1
```

## 启用端口转发内核模块
```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p
```

## 防火墙配置
```bash
firewall-cmd --permanent --add-masquerade
firewall-cmd --permanent --add-service=openvpn
firewall-cmd --permanent --add-port=11940/tcp
firewall-cmd --permanent --direct --passthrough ipv4 -t nat -A POSTROUTING -s 10.1.1.0/24 -o eth0 -j MASQUERADE
firewall-cmd --reload

```

## 登录
![image-20210308150955494](https://wangyp.cf/assets/img/image-20210308150955494.png)