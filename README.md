# fabric-web

## 版本历史
 
### v1.1.1

* 新增`expect`，免去手动输入密码的烦恼；

### v1.1.2

* 脚本自动分发到各个服务器
* orderer增加kafka集群
* 账本存储使用couchdb

### v1.1.3

* 支持动态新增组织

## 使用步骤

我们假定您已经配置好了go、docker环境，并设置了相关环境变量。

如果没有，请参考：[>> 环境搭建](https://github.com/hutu92/fabric-samples-cn/blob/master/README.md)

请先使用如下命令为脚本增加执行权限：

```bash
root@vm10-249-0-4:~/fabric-web/fabric-ca# chmod +x *.sh
root@vm10-249-0-4:~/fabric-web/fabric-ca# chmod +x scripts/*.sh
root@vm10-249-0-4:~/fabric-web/fabric-ca# chmod +x scripts/eyfn/*.sh
```

### 一、网络拓扑

通过`fabric.config`定义网络拓扑结构。

> 💡 请确认`setup`节点的IP与第一个Peer组织的第一个peer节点一致，
> 因为`setup-bootstrap.sh`脚本会启动`run-fabric.sh`脚本，而`run-fabric.sh`这个脚本是基于该节点身份执行一系列操作的，
> 否则的话，在执行实例化链码时报`timeout`错误。

### 二、构建项目，打包分发脚本

```bash
./network_builder.sh
```

### 三、下载镜像

**在正式开始前，确保您在每个节点都已下载所需的fabric镜像**

可执行如下命令下载镜像:

```bash
./down-images.sh
```

### 四、配置hosts

为了避免网络访问不同的情况，请确保修改每台服务器的`/etc/host`，内容参见`build/host.config`。

### 五、启动CA服务

对于每一个组织都要启动一个rca和ica服务。

**确保先启动rca，在启动ica**。

##### (1). rca(root ca)

一个组织对应一个 **_root fabric-ca-server_**

启动指定组织`<ORG>`的 **_root fabric-ca-server_** 命令如下

```bash
./rca-bootstrap.sh <ORG>
```

脚本会在Root CA初始化时，在`/etc/hyperledger/fabric-ca`目录下生成`ca-cert.pem`证书，并将其拷贝为`/${DATA}/${ORG}-ca-cert.pem`。

##### (2). ica(intermediate ca)

一个组织对应一个 **_intermediate fabric-ca-server_**

启动指定组织`<ORG>`的 **_intermediate fabric-ca-server_** 命令如下

```bash
./ica-bootstrap.sh <ORG>
```

脚本会在Intermediate CA初始化时，在`/etc/hyperledger/fabric-ca`目录下生成`ca-chain.pem`证书，并将其拷贝为`/${DATA}/${ORG}-ca-chain.pem`。

>其它节点下列操作需要使用rca(`USE_INTERMEDIATE_CA`为`false`时)或者ica(`USE_INTERMEDIATE_CA`为true`时)根证书
     
 >
 >- 向CA服务端申请组织根证书时使用;
 >- 向CA服务端登记CA管理员身份时使用;
 >    
 >    之所以*登记CA管理员身份*，是因为需要使用CA管理员身份去注册Orderer和Peer相关用户实体。
 >    
 >    **_!!! 执行注册新用户实体的客户端必须已经通过登记认证，并且拥有足够的权限来进行注册 !!!_**
 >
 >- 向CA服务端登记 **_Orderer组织管理员身份和Peer组织管理员身份_**、**_Orderer节点身份和Peer节点身份_**，以及 **_Peer组织普通用户身份_** 时使用;
 > 
 > 等等。
 
 因此，
 
 - `USE_INTERMEDIATE_CA`为`false`，即未启用中间层CA时，**_需要将`/etc/hyperledger/fabric-ca/ca-cert.pem`拷贝到其它节点作为`CA_CHAINFILE`_**；
 - `USE_INTERMEDIATE_CA`为`true`，即启用中间层CA时，**_需要将`/etc/hyperledger/fabric-ca/ca-chain.pem`拷贝到其它节点作为`CA_CHAINFILE`_**;
 
 不必担心，这些工作脚本已经帮我们完成了！~ :laughing: 
 
 采用的方法是其它节点通过 **_ssh远程拷贝ca上的根证书_** ，所以我们在`fabric.config`中配置了CA的`USER_NAME`、`PWD`、`PATH`，
 此外，我们还通过`expect`免去了你与脚本交互（scp远程拷贝需要您输入服务器密码）。 
 
 同样的，
 
 - `orderer`节点需要从`setup`节点获取创世区块
 - `run`节点需要从`setup`节点获取应用通道配置交易文件、锚节点配置更新交易文件
 
 这些工作脚本也已经帮我们完成了！~ ✌ 
 
 ### 六、启动setup
 
 `setup`容器用于：
 
 - 向fabric-ca-servers注册Orderer和Peer身份
 - 构建通道Artifacts（包括：创世区块、应用通道配置交易文件、锚节点配置更新交易文件）
 - 启动`run`容器，执行创建应用通道、加入应用通道、更新锚节点、安装链码、实例化链码、查询调用链码等操作
 
 > 务必在执行完步骤四，再执行此步骤。确保已成功启动所有组织的`rca`、`ica`节点。
 
 ```text
 setup-bootstrap.sh [-h] [-d]
             -h|-?       获取此帮助信息
             -d          从网络下载二进制文件
 ```
 
 * 脚本需要使用fabric的二进制文件`configtxgen`，请将这些二进制文件置于PATH路径下。
 
     如果脚本找不到，会基于[fabric源码](https://github.com/hyperledger/fabric)自动编译生成二进制文件，
     此时需要保证`$HOME/gopath/src/github.com/hyperledger/fabric`源码存在，且版本一致。
     
     当然你也可以通过指定`-d`选项从网络下载该二进制文件，这可能会很慢，取决于你的网速。
 
 * ~~脚本需要使用fabric的二进制文件`fabric-ca-client`，请将该二进制文件置于PATH路径下。~~
 
     ~~如果脚本找不到，会基于[fabric ca源码](https://github.com/hyperledger/fabric-ca)自动编译生成二进制文件，
     此时需要保证`$HOME/gopath/src/github.com/hyperledger/fabric-ca`源码存在，且版本一致。~~
 
     ~~编译`fabric-ca`相关代码，需要一些依赖包，可以通过如下命令安装:~~
 
     ```bash
     sudo apt-get install libtool libltdl-dev
     ```
 
     ~~脚本会将编译生成的`fabric-ca-server`和`fabric-ca-client`保存在`$GOPATH/bin`目录下。~~

 
 如果你执行完上述，那么来启动`setup`吧！~😍
 
 > 由于`setup-bootstrap.sh`脚本的一些操作需要较高的权限，请使用`root`用户运行。
 
 ```bash
 ./setup-bootstrap.sh
 ```
 
 ### 七、启动Zookeeper与Kafka集群
 
 ```text
 zk-kafka-bootstrap.sh <-z|-k> <ID>
     -h|-?       获取此帮助信息
     -z          启动zookeeper节点
     -k          启动kafka节点
     <ID>        节点的编号
 ```
 
 假设zookeeper与kafka个配置3台。那么启动脚本如下：
 
 ##### (1). 启动Zookeeper
 
 ```bash
 ./zk-kafka-bootstrap.sh -z 1
 ./zk-kafka-bootstrap.sh -z 2
 ./zk-kafka-bootstrap.sh -z 3
 ```
 
 ##### (2). 启动Kafka
 
 ```bash
 ./zk-kafka-bootstrap.sh -k 1
 ./zk-kafka-bootstrap.sh -k 2
 ./zk-kafka-bootstrap.sh -k 3
 ```
 
 ### 八、启动orderer
 
 ```text
 orderer-bootstrap.sh [-h] <ORG> <NUM>
     -h|-?  - 获取此帮助信息
     <ORG>   - 启动的orderer组织名称
     <NUM>   - 启动的orderer组织的第几个节点
 ```
 
 ```bash
 ./orderer-bootstrap.sh <ORG> <NUM>
 ```
 
 ### 九、启动peer
 
 ```text
 peer-bootstrap.sh [-h] <ORG> <NUM>
     -h|-?  - 获取此帮助信息
     <ORG>   - 启动的peer组织的名称
     <NUM>   - 启动的peer组织的节点索引
 ```
 
 ```bash
 ./peer-bootstrap.sh <ORG> <NUM>
 ```
 
 ### 十、动态增加节点
 
[>> 动态增加节点](./EYFN.md)
 
 ## TODO
 
 * network_builder导致重写的env.sh，导致其不可再次复用；
 * 支持通过`fabric.config`配置是否启用中间层CA；
 * 动态新增组织，启动Peer调用链码验证后，应恢复数据至初始状态，以为后续启动多个节点做准备；