# FISCO BCOS 2.0+ 企业级部署多服务器节点
* [官方文档 ](https://fisco-bcos-documentation.readthedocs.io/zh_CN/latest/docs/enterprise_tools/tutorial_detail_operation.html)
## 部署4节点2机构2群组的组网模式
## 下载安装

**下载**

```bash
cd ~/ && git clone https://github.com/FISCO-BCOS/generator.git
```

**安装**

此操作要求用户具有sudo权限。

```bash
cd ~/generator && bash ./scripts/install.sh
```

检查是否安装成功，若成功，输出 usage: generator xxx

```bash
./generator -h
```

**获取节点二进制**

拉取最新fisco-bcos二进制文件到meta中

```bash
./generator --download_fisco ./meta
```

**检查二进制版本**

若成功，输出 FISCO-BCOS Version : x.x.x-x

```bash
./meta/fisco-bcos -v
```

### 机器环境

每个节点的IP，端口号为如下：

| 机构  | 节点  | 所属群组  | P2P地址           | RPC/channel监听地址|
| --- | --- | ----- | --------------- | --------------------- |
| 机构A | 节点0 | 群组1、2 | 192.168.0.44:30300 | 192.168.0.44:8545/:20200 |
|     | 节点1 | 群组1、2 | 192.168.0.44:30301 | 192.168.0.44:8546/:20201 |
| 机构B | 节点2 | 群组1、2   | 192.168.0.41:30300 | 192.168.0.41:8545/:20200 |
|     | 节点3 | 群组1、2   | 192.168.0.41:30301  | 192.168.0.41:8546/:20201 |

```eval_rst
.. important::

    针对云服务器中的vps服务器，RPC监听地址需要写网卡中的真实地址(如内网地址或127.0.0.1)，可能与用户登录的ssh服务器不一致。
```

### 涉及机构

搭链操作涉及多个机构的合作，包括：

-   证书颁发机构
-   搭建节点的机构（简称“机构”）

### 关键流程

本流程简要的给出**证书颁发机构**，**节点机构间**如何相互配合搭建区块链。

#### 一、初始化链证书

1.  证书颁发机构操作：
    -   生成链证书

#### 二、生成群组1

1.  证书颁发机构操作：颁发机构证书
    -   生成机构证书
    -   发送证书
2.  机构间独立操作
    -   修改配置文件`node_deployment.ini`
    -   生成节点证书及节点P2P端口地址文件
3.  选取其中一个机构为群组生成创世块
    -   收集群组内所有节点证书
    -   修改配置文件`group_genesis.ini`
    -   为群组生成创世块文件
    -   分发创世块文件
4.  机构间独立操作：生成节点
    -   收集群组其他节点的P2P端口地址文件
    -   生成节点
    -   启动节点

#### 三、初始化新机构

1.  证书颁发机构操作：颁发新机构证书
    -   生成机构证书
    -   发送证书

#### 四、生成群组2

1.  新机构独立操作
    -   修改配置文件`node_deployment.ini`
    -   生成节点证书及节点P2P端口地址文件
2.  选取其中一个机构为群组生成创世块
    -   收集群组内所有节点证书
    -   修改配置文件`group_genesis.ini`
    -   为群组生成创世块文件
    -   分发创世块文件
3.  新机构独立操作：生成节点
    -   收集群组其他节点的P2P端口地址文件
    -   生成节点
    -   启动节点
4.  已有机构操作：配置新群组
    -   收集群组其他节点的P2P端口地址文件
    -   配置新群组与新增节点的P2P端口地址
    -   重启节点

### 机构初始化

我们以**教程中下载的generator作为证书颁发机构**。

**初始化机构A**

```bash
cp -r ~/generator ~/generator-A
```

**初始化机构B**

```bash
cp -r ~/generator ~/generator-B
```

### 初始化链证书

在证书颁发机构上进行操作，一条联盟链拥有唯一的链证书ca.crt

用 `--generate_chain_certificate` 命令生成链证书

在证书生成机构目录下操作:

```bash
cd ~/generator
```

```bash
./generator --generate_chain_certificate ./dir_chain_ca
```

查看链证书及私钥:

```bash
ls ./dir_chain_ca
```

```bash
# 上述命令解释
# 从左至右分别为链证书、链私钥
ca.crt  ca.key
```

## 机构A、B构建群组1

### 初始化机构A

教程中为了简化操作直接生成了机构证书和私钥，实际应用时应该由机构本地生成私钥`agency.key`，再生成证书请求文件，向证书签发机构获取机构证书`agency.crt`。

在证书生成机构目录下操作:

```bash
cd ~/generator
```

生成机构A证书：

```bash
./generator --generate_agency_certificate ./dir_agency_ca ./dir_chain_ca agencyA
```

查看机构证书及私钥:

```bash
ls dir_agency_ca/agencyA/
```

```bash
# 上述命令解释
# 从左至右分别为机构证书、机构私钥、链证书
agency.crt  agency.key  ca.crt
```

发送链证书、机构证书、机构私钥至机构A，示例是通过文件拷贝的方式，从证书授权机构将机构证书发送给对应的机构，放到机构的工作目录的meta子目录下

```bash
cp ./dir_agency_ca/agencyA/* ~/generator-A/meta/
```

### 初始化机构B

在证书生成机构目录下操作:

```bash
cd ~/generator
```

生成机构B证书：

```bash
./generator --generate_agency_certificate ./dir_agency_ca ./dir_chain_ca agencyB
```

发送链证书、机构证书、机构私钥至机构B，示例是通过文件拷贝的方式，从证书授权机构将机构证书发送给对应的机构，放到机构的工作目录的meta子目录下

```bash
cp ./dir_agency_ca/agencyB/* ~/generator-B/meta/
```

```eval_rst
.. important::

    一条联盟链中只能用到一个根证书ca.crt，多服务器部署时不要生成多个根证书和私钥。一个群组只能有一个群组创世区块group.x.genesis
```

### 机构A修改配置文件

`node_deployment.ini` 为节点配置文件，企业级部署工具会根据`node_deployment.ini`下的配置生成相关节点证书，及生成节点配置文件夹等。

机构A修改conf文件夹下的`node_deployment.ini`如下所示:

在~/generator-A目录下执行下述命令

```bash
cd ~/generator-A
```

```bash
cat > ./conf/node_deployment.ini << EOF
[group]
group_id=1

[node0]
; host ip for the communication among peers.
; Please use your ssh login ip.
p2p_ip=192.168.0.44
; listen ip for the communication between sdk clients.
; This ip is the same as p2p_ip for physical host.
; But for virtual host e.g. vps servers, it is usually different from p2p_ip.
; You can check accessible addresses of your network card.
; Please see https://tecadmin.net/check-ip-address-ubuntu-18-04-desktop/
; for more instructions.
rpc_ip=192.168.0.44
p2p_listen_port=30300
channel_listen_port=20200
jsonrpc_listen_port=8545

[node1]
p2p_ip=192.168.0.44
rpc_ip=192.168.0.44
p2p_listen_port=30301
channel_listen_port=20201
jsonrpc_listen_port=8546
EOF
```

### 机构B修改配置文件

机构B修改conf文件夹下的`node_deployment.ini`如下图所示:

在~/generator-B目录下执行下述命令

```bash
cd ~/generator-B
```

```bash
cat > ./conf/node_deployment.ini << EOF
[group]
group_id=1

[node0]
; host ip for the communication among peers.
; Please use your ssh login ip.
p2p_ip=192.168.0.41
; listen ip for the communication between sdk clients.
; This ip is the same as p2p_ip for physical host.
; But for virtual host e.g. vps servers, it is usually different from p2p_ip.
; You can check accessible addresses of your network card.
; Please see https://tecadmin.net/check-ip-address-ubuntu-18-04-desktop/
; for more instructions.
rpc_ip=192.168.0.41
p2p_listen_port=30300
channel_listen_port=20200
jsonrpc_listen_port=8545

[node1]
p2p_ip=192.168.0.41
rpc_ip=192.168.0.44
p2p_listen_port=30301
channel_listen_port=20201
jsonrpc_listen_port=8546
EOF
```

### 机构A生成并发送节点信息

在~/generator-A目录下执行下述命令

```bash
cd ~/generator-A
```

机构A生成节点证书及P2P连接信息文件，此步需要用到上述配置的`node_deployment.ini`，及机构meta文件夹下的机构证书与私钥，机构A生成节点证书及P2P连接信息文件

```bash
./generator --generate_all_certificates ./agencyA_node_info
```

查看生成文件:

```bash
ls ./agencyA_node_info
```

```bash
# 上述命令解释
# 从左至右分别为需要交互给机构A的节点证书，节点P2P连接地址文件(根据node_deployment.ini生成的本机构节点信息)
cert_192.168.0.44_30300.crt cert_192.168.0.44_30301.crt peers.txt
```
机构生成节点时需要指定其他节点的节点P2P连接地址，因此，A机构需将节点P2P连接地址文件发送至机构B

```bash
cp ./agencyA_node_info/peers.txt ~/generator-B/meta/peersA.txt
```

### 机构B生成并发送节点信息

在~/generator-B目录下执行下述命令

```bash
cd ~/generator-B
```

机构B生成节点证书及P2P连接信息文件：

```bash
./generator --generate_all_certificates ./agencyB_node_info
```

生成创世区块的机构需要节点证书，示例中由A机构生成创世区块，因此B机构除了发送节点P2P连接地址文件外，还需发送节点证书至机构A

发送证书

```bash
cp ./agencyB_node_info/cert*.crt ~/generator-A/meta/
```

发送节点P2P连接地址文件

```bash
cp ./agencyB_node_info/peers.txt ~/generator-A/meta/peersB.txt
```

### 机构A生成群组1创世区块

在~/generator-A目录下执行下述命令

```bash
cd ~/generator-A
```

机构A修改conf文件夹下的`group_genesis.ini`。:

```bash
cat > ./conf/group_genesis.ini << EOF
[group]
group_id=1

[nodes]
node0=192.168.0.44:30300
node1=192.168.0.44:30301
node2=192.168.0.41:30300
node3=192.168.0.41:30301
EOF
```

命令执行之后会修改./conf/group_genesis.ini文件：

```ini
;命令解释
[group]
;群组id
group_id=1

[nodes]
;机构A节点p2p地址
node0=192.168.0.44:30300
;机构A节点p2p地址
node1=192.168.0.44:30301
;机构B节点p2p地址
node2=192.168.0.41:30300
;机构B节点p2p地址
node3=192.168.0.41:30301
```

教程中选择机构A生成群组创世区块，实际生产中可以通过联盟链委员会协商选择。

此步会根据机构A的meta文件夹下配置的节点证书，生成group_genesis.ini配置的群组创世区块，教程中需要机构A的meta下有名为`cert_192.168.0.44_30300.crt`，`cert_192.168.0.44_30301.crt`，`cert_192.168.0.41_30300.crt`，`cert_192.168.0.41_30301.crt`的节点证书，此步需要用到机构B的节点证书。

```bash
./generator --create_group_genesis ./group
```

分发群组1创世区块至机构B：

```bash
cp ./group/group.1.genesis ~/generator-B/meta
```

### 机构A生成所属节点

在~/generator-A目录下执行下述命令

```bash
cd ~/generator-A
```

生成机构A所属节点，此命令会根据用户配置的`node_deployment.ini`文件生成相应的节点配置文件夹:

注意，此步指定的节点P2P连接信息`peers.txt`为群组内其他节点的链接信息，多个机构组网的情况下需要将其合并。

```bash
./generator --build_install_package ./meta/peersB.txt ./nodeA
```

查看生成节点配置文件夹：

```bash
ls ./nodeA
```

```bash
# 命令解释 此处采用tree风格显示
# 生成的文件夹nodeA信息如下所示，
├── monitor # monitor脚本
├── node_192.168.0.44_30300 # 192.168.0.44服务器 端口号30300的节点配置文件夹
├── node_192.168.0.44_30301
├── scripts # 节点的相关工具脚本
├── start_all.sh # 节点批量启动脚本
└── stop_all.sh # 节点批量停止脚本
```
机构A启动节点：

```bash
bash ./nodeA/start_all.sh
```

查看节点进程:

```bash
ps -ef | grep fisco
```

```bash
# 命令解释
# 可以看到如下进程
fisco  15347     1  0 17:22 pts/2    00:00:00 ~/generator-A/nodeA/node_192.168.0.44_30300/fisco-bcos -c config.ini
fisco  15402     1  0 17:22 pts/2    00:00:00 ~/generator-A/nodeA/node_192.168.0.44_30301/fisco-bcos -c config.ini
```

### 机构B生成所属节点

在~/generator-B目录下执行下述命令

```bash
cd ~/generator-B
```

生成机构B所属节点，此命令会根据用户配置的`node_deployment.ini`文件生成相应的节点配置文件夹:

```bash
./generator --build_install_package ./meta/peersA.txt ./nodeB
```

机构B启动节点：
将生成的节点文件夹推送至对应服务器192.168.0.41
```bash
bash ./nodeB/start_all.sh
```

### 查看群组1节点运行状态

查看进程：

```bash
ps -ef | grep fisco
```

```bash
# 命令解释
# 可以看到如下所示的进程
fisco  15347     1  0 17:22 pts/2    00:00:00 ~/generator-A/nodeA/node_192.168.0.44_30300/fisco-bcos -c config.ini
fisco  15402     1  0 17:22 pts/2    00:00:00 ~/generator-A/nodeA/node_192.168.0.44_30301/fisco-bcos -c config.ini

```

查看节点log：

```bash
tail -f ./node*/node*/log/log*  | grep +++
```

```bash
# 命令解释
# log中打印的+++即为节点正常共识
info|2019-02-25 17:25:56.028692| [g:1][p:264][CONSENSUS][SEALER]++++++++++++++++ Generating seal on,blkNum=1,tx=0,myIdx=0,hash=833bd983...
info|2019-02-25 17:25:59.058625| [g:1][p:264][CONSENSUS][SEALER]++++++++++++++++ Generating seal on,blkNum=1,tx=0,myIdx=0,hash=343b1141...
info|2019-02-25 17:25:57.038284| [g:1][p:264][CONSENSUS][SEALER]++++++++++++++++ Generating seal on,blkNum=1,tx=0,myIdx=1,hash=ea85c27b...
```

至此，我们完成了机构A、B搭建群组1的操作。

---

## 机构A、B构建群组2

接下来，机构B需要与A进行新群组建立操作

```bash
cd ~/generator-A
```

修改机构A生成文件群组id
```bash
vim conf/group_genesis.ini
[group]
group_id=2

[nodes]
node0=192.168.0.44:40300
node1=192.168.0.44:40301
node2=192.168.0.41:40300
node3=192.168.0.41:40301
```

创建群组2创世区块
```bash
./generator --create_group_genesis ./group
```

重启机构A节点:

```bash
bash ./nodeA/stop_all.sh
```

```bash
bash ./nodeA/start_all.sh
```

分发群组2创世区块至机构B：

```bash
scp ./group/group.2.genesis 192.168.0.41:~/generator-B/meta/
```

### 机构B为现有节点初始化群组2

在~/generator-B目录下执行下述命令

```bash
cd ~/generator-B
```

添加群组2配置文件至已有节点，此步将群组2创世区块`group.2.genesis`添加至./nodeB下的所有节点内：
报错请先安装 `pip install configparser`

```bash
./generator --add_group ./meta/group.2.genesis ./nodeB
```
重启机构B节点:

```bash
bash ./nodeB/stop_all.sh
```

```bash
bash ./nodeB/start_all.sh
```
