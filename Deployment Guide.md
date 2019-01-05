# 物理主机安装

## 安装ubuntu 16.04.3 LTS server版本

略

## 配置apt源

### 设置内部更新源

由于后期环境将不能连接互联网更新软件，需要建立内部的更新源，参考[这里](https://jingyan.baidu.com/article/90895e0f3e10d264ed6b0b78.html)和[这里](https://www.cnblogs.com/xwdreamer/p/3875857.html)

安装`apt-mirror`和`apache2`

~~~shell
sudo apt-get install -y apt-mirror apache2
~~~

编辑`/etc/apt/mirror.list`，修改更新源地址为阿里云镜像地址（不要指定deb-i386，因为你的server不一定是intel 32bit的）

~~~shell
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse 
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse 
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse 
deb http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse 
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse 
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse 
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse 
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse 
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse 
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse 
~~~

将mirror拉取到本地，默认在`/var/spool/apt-mirror/`文件夹下

~~~shell
sudo apt-mirror
~~~

将apach2的html的地址指向mirror的存储目录，启动web服务

~~~shell
sudo ln -s /var/spool/apt-mirror/mirror/mirrors.aliyun.com/ubuntu /var/www/html/ubuntu
sudo service apache2 start
~~~

### 配置源

在所有非master节点上，编辑`/etc/apt/source.list`，将源修改为私有源

~~~shell
deb http://192.168.137.100/ubuntu xenial main restricted universe multiverse
~~~

之后执行

~~~bash
sudo apt-get update
~~~

## 安装SSHserver

~~~bash
sudo apt-get install openssh-server
~~~

## 安装4.10.0-041000-lowlatency的内核

### 安装内核

可以用过`uname -a`查看目前内核版本

~~~bash
lab931@tnlist-wks2:~$ uname -a
Linux tnlist-wks2 4.8.0-59-generic #64-Ubuntu SMP Thu Jun 29 19:38:34 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
~~~

> 4.4的内核只能跑eNodeB不能跑EPC，建议沿用我们文档中4.10的内核版本

打开[4.10内核下载链接](https://kernel.ubuntu.com/~kernel-ppa/mainline/v4.10/)，下载以下三个安装包

~~~
Build for amd64 succeeded (see BUILD.LOG.amd64):
  linux-headers-4.10.0-041000_4.10.0-041000.201702191831_all.deb
  linux-headers-4.10.0-041000-lowlatency_4.10.0-041000.201702191831_amd64.deb
  linux-image-4.10.0-041000-lowlatency_4.10.0-041000.201702191831_amd64.deb
~~~

下载完毕之后执行

~~~bash
lab931@tnlist-wks2:~$ dpkg -i linux-headers-4.10.0-041000_4.10.0-041000.201702191831_all.deb linux-headers-4.10.0-041000-lowlatency_4.10.0-041000.201702191831_amd64.deb linux-image-4.10.0-041000-lowlatency_4.10.0-041000.201702191831_amd64.deb
~~~

 ### 更新GUB配置

安装之后更新GRUB配置，确保重新启动后是低延时内核。更改`/boot/gurb/gurb.cfg`的配置

~~~shell
# 启动项的配置格式为第一个menuentry为默认项，接下来submenu是可选项，submenu中也有大量的menuentry为不同的内核版本，按照从高到低顺序排列
menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-5c844bf7-aeff-4214-a1bf-d7f51500f97c' 
submenu 'Advanced options for Ubuntu' $menuentry_id_option 'gnulinux-advanced-5c844bf7-aeff-4214-a1bf-d7f51500f97c' {
	menuentry 'Ubuntu, with Linux 4.15.0-29-generic' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-4.15.0-29-generic-advanced-5c844bf7-aeff-4214-a1bf-d7f51500f97c'
	# 找到submenu中的4.10.0-041000-lowlatency的选项，覆盖默认的menuentry即可
~~~

最后更新一次GRUB配置

~~~bash
sudo update-grub
~~~

最后重启系统，会默认进入4.10-lowlatency的内核

## 安装网络工具

~~~bash
sudo apt-get install bridge-utils
~~~

## 设定网卡编号规则

修改`/etc/default/grub`，添加一行，使得网卡从`eth0`开始编号

~~~shell
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
~~~

修改`/etc/network/interface`

~~~shell
# 找到eth0的配置信息，IP address自行修改
# The primary network interface
auto eth0
iface eth0 inet static
address 192.168.137.100
netmask 255.255.255.0
gateway 192.168.137.1
dns-nameservers 192.168.137.1

auto eth1
iface eth1 inet static
address 192.168.1.100
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 192.168.1.1
~~~

## 修改主机主板配置

按照[OAI官网教程](https://gitlab.eurecom.fr/oai/openairinterface5g/wikis/OpenAirKernelMainSetup)，进入BIOS，关闭所有的电源选项，包括`sleep states`, `C-states`和`CPU frequency scaling`选项，以及关闭`hyperthreading`选项

1. 修改`/etc/default/grub`，添加**`GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_pstate=disable"`**和**`GRUB_CMDLINE_LINUX_DEFAULT="processor.max_cstate=1"`**以及**`GRUB_CMDLINE_LINUX_DEFAULT="intel_idle.max_cstate=0"`**以及**`GRUB_CMDLINE_LINUX_DEFAULT="idle=poll"`**，然后`update-grub`

2. 修改`/etc/modprobe.d/blacklist.conf`，如果文件不存在就创建一个，加一行**`blacklist intel_powerclamp`**

3. 安装`i7z`检查CPU设置

   ~~~bash
   sudo apt-get install i7z
   sudo i7z
   ~~~

   CPU频率变化不能超过1~2赫兹，除了C0没有别的C-state，说明CPU设置完毕

   ![i7z_log](Capture\\i7z_log.png)

4. 安装`cpufrequtils`

   ~~~shell
   sudo apt-get isntall cpufrequtils
   ~~~

   修改`/etc/default/cpufrequtils`，加入

   ~~~shell
   GOVERNOR="performance"
   ~~~

   然后

   ~~~shell
   sudo update-rc.d ondemand disable
   sudo /etc/init.d/cpufrequtils restart
   ~~~

   可以输入`cpufreq-info`查看相关信息

## 安装docker

参照[官网安装文档](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

~~~bash
# 1. 安装必要软件
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# 2. 安装GPG证书
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu    $(lsb_release -cs)    stable"
# 3. 查看docker版本，查看是否有17.03.2~ce-0~ubuntu-xenial版本
sudo apt-cache madison docker-ce
# 4. 安装17.03.2版本的docker
sudo apt-get install docker-ce=17.03.2~ce-0~ubuntu-xenial
~~~

## 关闭防火墙

~~~shell
sudo ufw disable
~~~

## 设置时间同步

以master为时间同步服务器，作为集群的时间基准。

~~~shell
sudo apt-get install ntp
~~~

`service --status-all` 查看所有服务的状态

TODO: @Wen Xiang

# 集群规划

## 拓扑结构

![system topology](C:\Users\w\Desktop\WeStoneTestbed\Capture\system topology.jpg)

整个集群分为三个部分

* 管理服务器master
* 开发调试环境
  * 核心网服务器node3
  * 接入网服务器node0
* 测试环境
  * 终端区
    * OAI-UE终端一台
    * USRP两块
    * Q7四块
    * 商用UE两块
  * 接入网服务器node1
  * 核心网服务器node4

网络分为两部分

* 管理网络
  * 负责外部用户的登录
  * 负责控制面信息的传输
* 承载网络
  * 负责业务间通信

## 规划域名

设定本地的域名为`open5g.local`，六台服务器的域名地址分别为`master.open5g.local`，`node0~node5.open5g.local`

| hostname |    管理网eth0     |   承载网eth1    |    role    | 光口 |
| :------: | :---------------: | :-------------: | :--------: | :--: |
| `master` | `192.168.137.100` | `192.168.1.100` | controller |      |
| `node0`  | `192.168.137.200` | `192.168.1.200` |   OAISIM   |      |
| `node1`  | `192.168.137.201` | `192.168.1.201` |    eNB     |  1   |
| `node2`  | `192.168.137.202` | `192.168.1.202` |    eNB     |  1   |
| `node3`  | `192.168.137.203` | `192.168.1.203` |    EPC     |      |
| `node4`  | `192.168.137.204` | `192.168.1.204` |    EPC     |      |

## 设置docker镜像仓库

### master配置镜像仓库服务

参考[官网部署说明](https://docs.docker.com/registry/)

在master节点上启动镜像服务，服务监听在localhost:5000上，第一次会自动获取镜像

~~~shell
docker run -d -p 5000:5000 --restart=always --name registry registry:2
~~~

要将镜像上传到仓库，首先要修改tag，加上`192.168.137.100:5000/`前缀，例如上传ubuntu:16.04镜像

~~~shell
docker tag ubuntu:16.04 192.168.137.100:5000/ubuntu:16.04
docker push 192.168.137.100:5000/ubuntu:16.04
~~~

> 由于registry默认采用HTTPS访问，而这里没有配置TLS相关的key和crt文件，可以手动配置TLS，也可以部署insecure registry

参考[这里](https://docs.docker.com/registry/insecure/#deploy-a-plain-http-registry)部署insecure registry

编辑`/etc/docker/daemon.json`，加入

~~~shell
"insecure-registries" : ["192.168.137.100:5000"]
~~~

之后通过`service docker restart`重启docker服务即可

### 其余node的配置

所有node都要在`/etc/docker/daemon.json`中添加

```shell
{
	"insecure-registries" : ["192.168.137.100:5000"]
}
```

并重启docker才能使用镜像仓库

~~~
docker pull 192.168.137.100:5000/image_name:version
~~~

## 配置kubernetes集群（Deprecated）

### 安装kubernetes

利用阿里云的镜像安装1.12.3-00版本的kubernetes

~~~shell
sudo apt-get update && apt-get install -y apt-transport-https
sudo curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF  
apt-get update
sudo apt-get install -y --allow-downgrades    kubelet=1.12.3-00 kubeadm=1.12.3-00 kubectl=1.12.3-00
~~~

TODO: @Xia Lei, @Liu Shasha

### master

### node

## swarm集群配置

### master节点

执行

~~~shell
docker swarm init --advertise-addr=192.168.137.100 --data-path-addr=192.168.1.100
~~~

`advertise-addr`是控制面通道，`data-path-addr`是数据面通道。屏幕会生成token，类似

~~~shell
To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.137.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
~~~

### node节点

执行

~~~shell
docker swarm join --advertise-addr=192.168.137.200 --data-path-addr=192.168.1.200 --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c 192.168.137.100:2377
~~~

token对应master上生成的token

### overlay网络搭建

在master节点上执行

~~~shell
docker network create -d overlay --subnet=10.244.100.0/24 oai-net1
~~~

而在各个node节点上，会自动部署`oai-ne1`这个overlay网络

### 服务部署

在master节点执行

~~~shell
docker service create --network oai-net1 --name=test --tty image:tag /bin/bash
~~~

创建了一个指定镜像的业务，其中初始化一个`shell`，`--tty`一定要加上

之后，在各个容器中，已经有两张网卡，一张属于overlay网络，一张通过`docker_gwbridge`桥接到外部，是集群内部和集群外面的流量出口。

在eNodeB节点上，用`brctl`命令将`docker_gwbridge`绑定到光口上，就不用可以设置其他网络了。

### 监控

master节点执行

~~~shell
docker volume create portainer_data
docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
~~~

之后再浏览器登录`http://192.168.137.100:9000`，第一次要选择`connect to local`，进行登录（第一次要设置密码）。在`Home`界面点击local节点，然后在侧边栏就会看到swarm的相关信息

# OAI环境安装

## 拓扑结构

![OAI拓扑结构](Capture\\OAI topology.PNG)

## EPC

### 镜像与环境配置

在宿主机上执行

~~~shell
# 拉取镜像
docker pull epc_with_msgbus:latest epc_hss:latest
# 创建MME+SPGW容器
docker run -tid --net=oai-net1 --ip=10.244.0.3 -P --privileged -h lab931 -v /lib/modules:/lib/modules -v /usr/src/linux-headers:/usr/src/linux-headers --name=oai_mme_spgw epc_with_msgbus:latest /bin/bash
# 创建HSS容器
docker run -tid --net=oai-net1 --ip=10.244.0.4 -P --privileged -h lab931 --name=oai_hss epc_hss:latest /bin/bash
~~~

### EPC配置

`docker attach oai_mme_spgw`进入MME+SPGW的容器内部，修改如下文件

1. `/usr/local/etc/oai/msgbusDsp.conf`

   1. 将`MME_INTERFACE_FOR_S1_MME`修改为容器的IP和网卡名称
   2. 将`SGW_IPV4_FOR_S1U`修改为容器的IP和网卡名称，端口不变

   ~~~shell
   MME : 
   {
   	# ..............................................
   	NETWORK_INTERFACES : 
       {
           # MME binded interface for S1-C or S1-MME  communication (S1AP), can be ethernet interface, virtual ethernet interface, we don't advise wireless interfaces
           MME_INTERFACE_NAME_FOR_S1_MME         = "enp1s0f0";                         # YOUR NETWORK CONFIG HERE
           MME_IPV4_ADDRESS_FOR_S1_MME           = "172.16.1.108/24";             # YOUR NETWORK CONFIG HERE
   
   S-GW : 
   {
       NETWORK_INTERFACES : 
       {
       	# ................................
           # S-GW binded interface for S1-U communication (GTPV1-U) can be ethernet interface, virtual ethernet interface, we don't advise wireless interfaces
           SGW_INTERFACE_NAME_FOR_S1U_S12_S4_UP    = "enp1s0f0";                       # STRING, interface name, YOUR NETWORK CONFIG HERE, USE "lo" if S-GW run on eNB host
           SGW_IPV4_ADDRESS_FOR_S1U_S12_S4_UP      = "172.16.1.108/24";           # STRING, CIDR, YOUR NETWORK CONFIG HERE
           SGW_IPV4_PORT_FOR_S1U_S12_S4_UP         = 2152;                         # INTEGER, port number, PREFER NOT CHANGE UNLESS YOU KNOW WHAT YOU ARE DOING
   
   ~~~

2. `/usr/local/etc/oai/mme.conf`

   参照刚才(1)的改法修改`MME_FOR_S1`的参数

<!--这里仍然有bug，建议两个文件都修改-->

`docker attach oai_hss`进入HSS容器内部，修改`/usr/local/etc/oai/freeDiameter/mme_fd.conf`，找到`ConnectPeer`这一行，将其中`ConnectTo`字段修改为HSS容器的IP

~~~shell
# ConnectPeer
# Declare a remote peer to which this peer must maintain a connection. 
# In addition, this allows specifying non-default parameters for this peer only
# (for example disable SCTP with this peer, or use RFC3588-flavour TLS). 
# Note that by default, if a peer is not listed as a ConnectPeer entry, an 
# incoming connection from this peer will be rejected. If you want to accept 
# incoming connections from other peers, see the acl_wl.fdx? extension which 
# allows exactly this.

ConnectPeer= "hss.openair4G.eur" { ConnectTo = "127.0.0.1"; No_SCTP ; No_IPv6; Prefer_TCP; No_TLS; port = 3868;  realm = "openair4G.eur";};
~~~

之后仍然在HSS容器内部，执行

~~~shell
service mysql start
~~~

启动MySQL服务

### 编译与启动

在HSS容器里

~~~shell
cd /epc/scripts
./build_hss		# 没有代码改动不用编译
./run_hss
~~~

在MME+SPGW容器里

~~~shell
cd /epc/scripts
# 如果对代码有所改动
./build_epcWithMsgBus
# 如果只改动配置文件，则不用编译
./run_epcWithMsgBus
~~~

## eNodeB

### 镜像与环境配置

宿主机上执行

~~~shell
# 获取镜像
docker pull oai_enb:yangzhou
# eNodeB容器需要一个网卡与硬件连接，在宿主机创建一个名为br2的网桥
docker network create --subnet=10.245.0.0/24 --opt com.docker.network.bridge.name=br2 oai-net2
# 将br2与服务器光口绑定
sudo brctl addif eth3 br2
# 创建eNB容器
docker run -tid --net=oai-net1 --ip=10.244.0.100 --privileged --name=eNB0 oai_enb:yangzhou bash
# 在容器中添加虚拟网卡，加入oai-net2
docker network connect oai-net2 eNB0
~~~

### eNodeB配置

在eNB容器中修改`~/OAI/openairinterface5g/targets/PROJECTS/GENERIC-LTE-EPC/CONF/enb.band7.tm1.usrpb210.conf`

* 将`mme_ip_address`改成MME+SPGW容器的IP
* 将`ENB_INTERFACE_FOR_S1`改为eNB容器的网卡和名称

~~~shell
////////// MME parameters:
    mme_ip_address      = ( { ipv4       = "172.19.1.3";	# YOUR CONFIG HERE
                              ipv6       = "192:168:30::17";
                              active     = "yes";
                              preference = "ipv4";
                            }
                          );

    NETWORK_INTERFACES :
    {
        ENB_INTERFACE_NAME_FOR_S1_MME            = "eth0";	# YOUR CONFIG HERE
        ENB_IPV4_ADDRESS_FOR_S1_MME    = "172.19.1.2/24";	# YOUR CONFIG HERE

        ENB_INTERFACE_NAME_FOR_S1U               = "eth0";	# YOUR CONFIG HERE
        ENB_IPV4_ADDRESS_FOR_S1U    = "172.19.1.2/24";		# YOUR CONFIG HERE
        ENB_PORT_FOR_S1U                         = 2152; # Spec 2152
    };

~~~

###编译与启动

在eNB容器中的根目录`/`下有脚本`start.sh`，如果需要编译，将如下两行uncomment

```shell
#cd $PATH_OAI
#./cmake_targets/build_oai -c --eNB -w USRP
```

之后执行

~~~shell
./start.sh 2150 01 10 run
~~~

## OAISIM

refer to [OAISIM部署文档](oaisim-deploy\README.md)

## OAI-UE

TODO: @Lu Jiegang