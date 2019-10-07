# CentOS 7部署 Ceph分布式存储架构

## 概述：
随着OpenStack日渐成为开源云计算的标准软件栈，Ceph也已经成为OpenStack的首选后端存储。Ceph是一种为优秀的性能、可靠性和可扩展性而设计的统一的、分布式文件系统。

ceph官方文档 http://docs.ceph.org.cn/

ceph中文开源社区 http://ceph.org.cn/

Ceph是一个开源的分布式文件系统。因为它还支持块存储、对象存储，所以很自然的被用做云计算框架openstack或cloudstack整个存储后端。当然也可以单独作为存储，例如部署一套集群作为对象存储、SAN存储、NAS存储等。

### ceph支持
1. 对象存储：即radosgw,兼容S3接口。通过rest api上传、下载文件。
2. 文件系统：posix接口。可以将ceph集群看做一个共享文件系统挂载到本地。
3. 块存储：即rbd。有kernel rbd和librbd两种使用方式。支持快照、克隆。相当于一块硬盘挂到本地，用法和用途和硬盘一样。比如在OpenStack项目里，Ceph的块设备存储可以对接OpenStack的后端存储

### Ceph相比其它分布式存储有哪些优点？
1. 统一存储

　　　　虽然ceph底层是一个分布式文件系统，但由于在上层开发了支持对象和块的接口。所以在开源存储软件中，能够一统江湖。至于能不能千秋万代，就不知了。

2. 高扩展性

　　　　扩容方便、容量大。能够管理上千台服务器、EB级的容量。

3. 可靠性强

　　　　支持多份强一致性副本，EC。副本能够垮主机、机架、机房、数据中心存放。所以安全可靠。存储节点可以自管理、自动修复。无单点故障，容错性强。

4. 高性能

　　　　因为是多个副本，因此在读写操作时候能够做到高度并行化。理论上，节点越多，整个集群的IOPS和吞吐量越高。另外一点ceph客户端读写数据直接与存储设备(osd) 交互。

### Ceph各组件介绍：
　　•Ceph OSDs: Ceph OSD 守护进程（ Ceph OSD ）的功能是存储数据，处理数据的复制、恢复、回填、再均衡，并通过检查其他OSD 守护进程的心跳来向 Ceph Monitors 提供一些监控信息。当 Ceph 存储集群设定为有2个副本时，至少需要2个 OSD 守护进程，集群才能达到 active+clean 状态（ Ceph 默认有3个副本，但你可以调整副本数）。

　　•Monitors:  Ceph Monitor维护着展示集群状态的各种图表，包括监视器图、 OSD 图、归置组（ PG ）图、和 CRUSH 图。 Ceph 保存着发生在Monitors 、 OSD 和 PG上的每一次状态变更的历史信息（称为 epoch ）。

　　•MDSs:  Ceph 元数据服务器（ MDS ）为 Ceph 文件系统存储元数据（也就是说，Ceph 块设备和 Ceph 对象存储不使用MDS ）。元数据服务器使得 POSIX 文件系统的用户们，可以在不对 Ceph 存储集群造成负担的前提下，执行诸如 ls、find 等基本命令。

## 实验集群部署
### 主机准备（禁用selinux，关闭防火墙）
每个节点两个网卡ens33和ens34

enss33作为cluster_network

enss34作为public_network

ens33 自动获取  ens34 设置静态IP ens34的IP地址作为管理地址

hostname	IP			Disk

ceph-node1  ens34 192.168.1.101  sda(OS) sdb sdc sdd

ceph-node2  ens34 192.168.1.102  sda(OS) sdb sdc sdd

ceph-node1  ens34 192.168.1.103  sda(OS) sdb sdc sdd

ceph-node1  ens34 192.168.1.104  sda(OS) sdb sdc sdd


### 编辑hosts文件
	[root@ceph-node1 ~]# cat /etc/hosts
	127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
	::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
	192.168.1.101 ceph-node1
	192.168.1.102 ceph-node2
	192.168.1.103 ceph-node3
	192.168.1.104 ceph-node4

	# ssh免密码登录
	# 在管理节点使用ssh-keygen生成ssh keys发布到各节点
	[root@ceph-node1 ~]# ssh-keygen 
	[root@ceph-node1 ~]# ssh-copy-id ceph-node1
	[root@ceph-node1 ~]# ssh-copy-id ceph-node2
	[root@ceph-node1 ~]# ssh-copy-id ceph-node3
	[root@ceph-node1 ~]# ssh-copy-id ceph-node4

### 管理节点安装ceph-deploy

#### 增加yum配置文件（各个节点都需要增加yum源）
##### 本实验选用mimic版本 为后续升级最新版本nautilus的实验做准备
  	[root@ceph-node1 ~]# vim /etc/yum.repos.d/ceph.repo
	# 添加以下内容：(ceph国内163源)
	[Ceph]
	name=Ceph packages for $basearch
	baseurl=http://mirrors.163.com/ceph/rpm-mimic/el7/x86_64
	enabled=1
	gpgcheck=1
	type=rpm-md
	gpgkey=http://mirrors.163.com/ceph/keys/release.asc
	priority=1
	
	[Ceph-noarch]
	name=Ceph noarch packages
	baseurl=http://mirrors.163.com/ceph/rpm-mimic/el7/noarch
	enabled=1
	gpgcheck=1
	type=rpm-md
	gpgkey=http://mirrors.163.com/ceph/keys/release.asc
	priority=1
	
	[ceph-source]
	name=Ceph source packages
	baseurl=http://mirrors.163.com/ceph/rpm-mimic/el7/SRPMS
	enabled=1
	gpgcheck=1
	type=rpm-md
	gpgkey=http://mirrors.163.com/ceph/keys/release.asc
	priority=1

	或者
	[Ceph]
	name=Ceph packages for $basearch
	baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/$basearch
	enabled=1
	gpgcheck=1
	type=rpm-md
	gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
	priority=1
	[Ceph-noarch]
	name=Ceph noarch packages
	baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch
	enabled=1
	gpgcheck=1
	type=rpm-md
	gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
	priority=1
	[ceph-source]
	name=Ceph source packages
	baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/SRPMS
	enabled=1
	gpgcheck=1
	type=rpm-md
	gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
	priority=1
#### 更新软件yum源并安装ceph-deploy管理工具 
	[root@ceph-node1 ceph]# yum clean all
	[root@ceph-node1 ceph]# yum makecache
	# 在ceph-node1节点上执行下面的命令安装ceph-deploy
	[root@ceph-node1 ~]# yum install -y ceph-deploy

### 创建monitor
	[root@ceph-node1 ~]# mkdir /etc/ceph
	[root@ceph-node1 ~]# cd /etc/ceph/
	[root@ceph-node1 ceph]# 
	# 使用ceph-deploy工具创建一个ceph集群。
	发现报错：
	[root@ceph-node1 ceph]# ceph-deploy new ceph-node1
	[root@ceph-node1 ceph]# ceph-deploy --version     
	Traceback (most recent call last):
	  File "/usr/bin/ceph-deploy", line 18, in <module>
	    from ceph_deploy.cli import main
	  File "/usr/lib/python2.7/site-packages/ceph_deploy/cli.py", line 1, in <module>
	    import pkg_resources
	ImportError: No module named pkg_resources
	原因是缺Python-setuptools 安装即可：
	[root@ceph-node1 ceph]# yum install -y python-setuptools 
	[root@ceph-node1 ceph]# ceph-deploy new ceph-node1
	
	# 生成配置文件在当前目录下
	[root@ceph-node1 ceph]# ll
	total 12
	-rw-r--r-- 1 root root  201 Sep 30 08:46 ceph.conf
	-rw-r--r-- 1 root root 3000 Sep 30 08:46 ceph-deploy-ceph.log
	-rw------- 1 root root   73 Sep 30 08:46 ceph.mon.keyring
	# ceph配置文件，一个monitor秘钥环和一个日志文件
	
### 修改副本数	
	# 配置文件的默认副本数是3 从3该成2 这样只有两个osd也能大刀片active+clean状态
	[root@ceph-node1 ceph]# vim ceph.conf 
	把这面这行加入到[global]段（可选配置 修改为2可减少一台几点 此处实验为4台服务器无需修改）
	[global]
	fsid = d77fa2a9-198e-4a9f-a106-8dbabe043e8d
	mon_initial_members = ceph-node1
	mon_host = 192.168.1.101
	auth_cluster_required = cephx
	auth_service_required = cephx
	auth_client_required = cephx
	osd_pool_default_size = 2
	
### 在所有节点安装ceph
	
	[root@ceph-node1 ceph]# ceph-deploy install ceph-node1 ceph-node2 ceph-node3 ceph-node4
	# 也可以使用 --release参数指定安装版本
	ceph-deploy install --release mimic ceph-node1 ceph-node2 ceph-node3 ceph-node4
	# 在任意节点检查Ceph版本以及ceph集群健康状态
	[root@ceph-node1 ceph]# ceph -v
	ceph version 13.2.6 (7b695f835b03642f85998b2ae7b6dd093d9fbce4) mimic (stable)
	# 在ceph-node1上创建第一个monitor
	[root@ceph-node1 ceph]# ceph-deploy mon create-initial
	# 检查集群状态  此阶段集群不会处于健康状态
	[root@ceph-node1 ceph]# ceph status
	  cluster:
	    id:     d77fa2a9-198e-4a9f-a106-8dbabe043e8d
	    health: HEALTH_OK
	 
	  services:
	    mon: 1 daemons, quorum ceph-node1
	    mgr: no daemons active
	    osd: 0 osds: 0 up, 0 in
	 
	  data:
	    pools:   0 pools, 0 pgs
	    objects: 0  objects, 0 B
	    usage:   0 B used, 0 B / 0 B avail
	    pgs: 


### 部署OSD
#### 执行下列步骤，在ceph-node1节点上创建一个存储设备（OSD）并将其加入Ceph集群中

	# 列出VM上的磁盘 
	# 在命令的输出中 确认要在哪些磁盘上创建 Ceph OSD（需要提出OS盘）
	# 在这里 这些磁盘名称应该是sdb、sdc、sdd
	[root@ceph-node1 ceph]# ceph-deploy disk list ceph-node1
	[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
	[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy disk list ceph-node1
	[ceph_deploy.cli][INFO  ] ceph-deploy options:
	[ceph_deploy.cli][INFO  ]  username                      : None
	[ceph_deploy.cli][INFO  ]  verbose                       : False
	[ceph_deploy.cli][INFO  ]  debug                         : False
	[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
	[ceph_deploy.cli][INFO  ]  subcommand                    : list
	[ceph_deploy.cli][INFO  ]  quiet                         : False
	[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fd5463184d0>
	[ceph_deploy.cli][INFO  ]  cluster                       : ceph
	[ceph_deploy.cli][INFO  ]  host                          : ['ceph-node1']
	[ceph_deploy.cli][INFO  ]  func                          : <function disk at 0x7fd546561938>
	[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
	[ceph_deploy.cli][INFO  ]  default_release               : False
	[ceph-node1][DEBUG ] connected to host: ceph-node1 
	[ceph-node1][DEBUG ] detect platform information from remote host
	[ceph-node1][DEBUG ] detect machine type
	[ceph-node1][DEBUG ] find the location of an executable
	[ceph-node1][INFO  ] Running command: fdisk -l
	[ceph-node1][INFO  ] Disk /dev/sda: 21.5 GB, 21474836480 bytes, 41943040 sectors
	[ceph-node1][INFO  ] Disk /dev/sdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
	[ceph-node1][INFO  ] Disk /dev/sdc: 10.7 GB, 10737418240 bytes, 20971520 sectors
	[ceph-node1][INFO  ] Disk /dev/sdd: 10.7 GB, 10737418240 bytes, 20971520 sectors
	[ceph-node1][INFO  ] Disk /dev/mapper/centos-root: 18.2 GB, 18249416704 bytes, 35643392 sectors
	[ceph-node1][INFO  ] Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors

#### 分发keying
我们执行admin的命令，要提供admin的key（–keyring ceph.client.admin.keyring）以及配置文件(-c ceph.conf)。在后续的运维中，我们经常需要在某个server上执行admin命令。每次都提供这些参数比较麻烦。实际上，ceph会默认地从/etc/ceph/中找keyring和ceph.conf。因此，我们可以把ceph.client.admin.keyring和ceph.conf放到每个server的/etc/ceph/。ceph-deploy可以帮我做这些：

	[root@ceph-node1 ceph]# ceph-deploy admin ceph-node1 ceph-node2 ceph-node3 ceph-node4
	[root@ceph-node1 ceph]# ceph -v
	ceph version 13.2.6 (7b695f835b03642f85998b2ae7b6dd093d9fbce4) mimic (stable)
	#检查每个server, /etc/ceph/下都多了ceph.client.admin.keying和ceph.conf这两个文件。
	[root@ceph-node1 ~]# ceph -s
	[root@ceph-node1 ~]# ceph auth get client.admin
	exported keyring for client.admin
	[client.admin]
	        key = AQDVXZFdiqETLBAAqlOX2mKvIY64DTJoGNNQWw==
	        caps mds = "allow *"
	        caps mgr = "allow *"
	        caps mon = "allow *"
	        caps osd = "allow *"

#### 创建mgr
ceph12(Lurminous)开始，需要为每个monitor创建一个mgr(其功能待研究，之前版本都有这个组件)

	[root@ceph-node1 ~]# ceph-deploy mgr create ceph-node1
	[root@ceph-node1 ~]# ceph -s
	  cluster:
	    id:     d77fa2a9-198e-4a9f-a106-8dbabe043e8d
	    health: HEALTH_OK
	 
	  services:
	    mon: 1 daemons, quorum ceph-node1
	    mgr: ceph-node1(active)
	    osd: 3 osds: 3 up, 3 in
	 
	  data:
	    pools:   0 pools, 0 pgs
	    objects: 0  objects, 0 B
	    usage:   3.0 GiB used, 27 GiB / 30 GiB avail
	    pgs:    

ceph-deploy 的disk zap命令会销毁磁盘中已存在的分区表和数据。运行这个命令前，未必确保使用争取的磁盘名称
	
	[root@ceph-node1 ceph]# ceph-deploy disk zap ceph-node1 /dev/sdb /dev/sdc /dev /sdd
	# 磁盘作block ，无block.db, 无block.wal
	[root@ceph-node1 ceph]# ceph-deploy osd create ceph-node1 --data /dev/sdb

	[root@ceph-node1 ceph]# ceph -s
	  cluster:
	    id:     d77fa2a9-198e-4a9f-a106-8dbabe043e8d
	    health: HEALTH_OK
	 
	  services:
	    mon: 1 daemons, quorum ceph-node1
	    mgr: ceph-node1(active)
	    osd: 3 osds: 3 up, 3 in
	 
	  data:
	    pools:   0 pools, 0 pgs
	    objects: 0  objects, 0 B
	    usage:   3.0 GiB used, 27 GiB / 30 GiB avail
	    pgs:     
	 
	[root@ceph-node1 ceph]# ceph -s
	  cluster:
	    id:     d77fa2a9-198e-4a9f-a106-8dbabe043e8d
	    health: HEALTH_OK
	 
	  services:
	    mon: 1 daemons, quorum ceph-node1
	    mgr: ceph-node1(active)
	    osd: 3 osds: 3 up, 3 in
	 
	  data:
	    pools:   0 pools, 0 pgs
	    objects: 0  objects, 0 B
	    usage:   3.0 GiB used, 27 GiB / 30 GiB avail
	    pgs: 


## 纵向扩展Ceph集群————添加monitor和OSD

### 添加monitor

	[root@ceph-node1 ceph]# ceph-deploy mon add ceph-node2    
	[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
	[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mon add ceph-node2
	[ceph_deploy.cli][INFO  ] ceph-deploy options:
	[ceph_deploy.cli][INFO  ]  username                      : None
	[ceph_deploy.cli][INFO  ]  verbose                       : False
	[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
	[ceph_deploy.cli][INFO  ]  subcommand                    : add
	[ceph_deploy.cli][INFO  ]  quiet                         : False
	[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f38e9484fc8>
	[ceph_deploy.cli][INFO  ]  cluster                       : ceph
	[ceph_deploy.cli][INFO  ]  mon                           : ['ceph-node2']
	[ceph_deploy.cli][INFO  ]  func                          : <function mon at 0x7f38e96e6410>
	[ceph_deploy.cli][INFO  ]  address                       : None
	[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
	[ceph_deploy.cli][INFO  ]  default_release               : False
	[ceph_deploy.mon][INFO  ] ensuring configuration of new mon host: ceph-node2
	[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-node2
	[ceph-node2][DEBUG ] connected to host: ceph-node2 
	[ceph-node2][DEBUG ] detect platform information from remote host
	[ceph-node2][DEBUG ] detect machine type
	[ceph-node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
	[ceph_deploy.mon][DEBUG ] Adding mon to cluster ceph, host ceph-node2
	[ceph_deploy.mon][DEBUG ] using mon address by resolving host: 192.168.1.102
	[ceph_deploy.mon][DEBUG ] detecting platform for host ceph-node2 ...
	[ceph-node2][DEBUG ] connected to host: ceph-node2 
	[ceph-node2][DEBUG ] detect platform information from remote host
	[ceph-node2][DEBUG ] detect machine type
	[ceph-node2][DEBUG ] find the location of an executable
	[ceph_deploy.mon][INFO  ] distro info: CentOS Linux 7.7.1908 Core
	[ceph-node2][DEBUG ] determining if provided host has same hostname in remote
	[ceph-node2][DEBUG ] get remote short hostname
	[ceph-node2][DEBUG ] adding mon to ceph-node2
	[ceph-node2][DEBUG ] get remote short hostname
	[ceph-node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
	[ceph-node2][DEBUG ] create the mon path if it does not exist
	[ceph-node2][DEBUG ] checking for done path: /var/lib/ceph/mon/ceph-ceph-node2/done
	[ceph-node2][DEBUG ] create a done file to avoid re-doing the mon deployment
	[ceph-node2][DEBUG ] create the init path if it does not exist
	[ceph-node2][INFO  ] Running command: systemctl enable ceph.target
	[ceph-node2][INFO  ] Running command: systemctl enable ceph-mon@ceph-node2
	[ceph-node2][INFO  ] Running command: systemctl start ceph-mon@ceph-node2
	[ceph-node2][WARNIN] No data was received after 7 seconds, disconnecting...
	[ceph-node2][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.ceph-node2.asok mon_status
	[ceph-node2][ERROR ] admin_socket: exception getting command descriptions: [Errno 2] No such file or directory
	[ceph-node2][WARNIN] ceph-node2 is not defined in `mon initial members`
	[ceph-node2][WARNIN] monitor ceph-node2 does not exist in monmap
	[ceph-node2][WARNIN] neither `public_addr` nor `public_network` keys are defined for monitors
	[ceph-node2][WARNIN] monitors may not be able to form quorum
	[ceph-node2][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.ceph-node2.asok mon_status
	[ceph-node2][ERROR ] admin_socket: exception getting command descriptions: [Errno 2] No such file or directory
	[ceph-node2][WARNIN] monitor: mon.ceph-node2, might not be running yet
	# 报如上错误 经查证 原来要在ceph.conf定义一下Pulic_network
	vim ceph.conf
	public_network = 192.168.1.101/24
	cluster_network = 192.168.98.0/24
	# 当更新完ceph.conf , 运行同样的命令会再报错
	[root@ceph-node1 ceph]# ceph-deploy mon add ceph-node2
	[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
	[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mon add ceph-node2
	[ceph_deploy.cli][INFO  ] ceph-deploy options:
	[ceph_deploy.cli][INFO  ]  username                      : None
	[ceph_deploy.cli][INFO  ]  verbose                       : False
	[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
	[ceph_deploy.cli][INFO  ]  subcommand                    : add
	[ceph_deploy.cli][INFO  ]  quiet                         : False
	[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f460bb80fc8>
	[ceph_deploy.cli][INFO  ]  cluster                       : ceph
	[ceph_deploy.cli][INFO  ]  mon                           : ['ceph-node2']
	[ceph_deploy.cli][INFO  ]  func                          : <function mon at 0x7f460bde2410>
	[ceph_deploy.cli][INFO  ]  address                       : None
	[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
	[ceph_deploy.cli][INFO  ]  default_release               : False
	[ceph_deploy.mon][INFO  ] ensuring configuration of new mon host: ceph-node2
	[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-node2
	[ceph-node2][DEBUG ] connected to host: ceph-node2 
	[ceph-node2][DEBUG ] detect platform information from remote host
	[ceph-node2][DEBUG ] detect machine type
	[ceph-node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
	[ceph_deploy.admin][ERROR ] RuntimeError: config file /etc/ceph/ceph.conf exists with different content; use --overwrite-conf to overwrite
	[ceph_deploy][ERROR ] GenericError: Failed to configure 1 admin hosts，
	# 原因是conf不同步所致，--overwrite的使用各不相同 通过-h出来的命令是
	
	[root@ceph-node1 ceph]# ceph-deploy --overwrite-conf config push  ceph-node1 ceph-node2 ceph-node3 ceph-node4

	# 然后再运行ceph-deploy mon add ceph-node2
	[root@ceph-node1 ceph]# ceph-deploy mon add ceph-node2
	[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
	[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy mon add ceph-node2
	[ceph_deploy.cli][INFO  ] ceph-deploy options:
	[ceph_deploy.cli][INFO  ]  username                      : None
	[ceph_deploy.cli][INFO  ]  verbose                       : False
	[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
	[ceph_deploy.cli][INFO  ]  subcommand                    : add
	[ceph_deploy.cli][INFO  ]  quiet                         : False
	[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7ff260a22fc8>
	[ceph_deploy.cli][INFO  ]  cluster                       : ceph
	[ceph_deploy.cli][INFO  ]  mon                           : ['ceph-node2']
	[ceph_deploy.cli][INFO  ]  func                          : <function mon at 0x7ff260c84410>
	[ceph_deploy.cli][INFO  ]  address                       : None
	[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
	[ceph_deploy.cli][INFO  ]  default_release               : False
	[ceph_deploy.mon][INFO  ] ensuring configuration of new mon host: ceph-node2
	[ceph_deploy.admin][DEBUG ] Pushing admin keys and conf to ceph-node2
	[ceph-node2][DEBUG ] connected to host: ceph-node2 
	[ceph-node2][DEBUG ] detect platform information from remote host
	[ceph-node2][DEBUG ] detect machine type
	[ceph-node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
	[ceph_deploy.mon][DEBUG ] Adding mon to cluster ceph, host ceph-node2
	[ceph_deploy.mon][DEBUG ] using mon address by resolving host: 192.168.1.102
	[ceph_deploy.mon][DEBUG ] detecting platform for host ceph-node2 ...
	[ceph-node2][DEBUG ] connected to host: ceph-node2 
	[ceph-node2][DEBUG ] detect platform information from remote host
	[ceph-node2][DEBUG ] detect machine type
	[ceph-node2][DEBUG ] find the location of an executable
	[ceph_deploy.mon][INFO  ] distro info: CentOS Linux 7.7.1908 Core
	[ceph-node2][DEBUG ] determining if provided host has same hostname in remote
	[ceph-node2][DEBUG ] get remote short hostname
	[ceph-node2][DEBUG ] adding mon to ceph-node2
	[ceph-node2][DEBUG ] get remote short hostname
	[ceph-node2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
	[ceph-node2][DEBUG ] create the mon path if it does not exist
	[ceph-node2][DEBUG ] checking for done path: /var/lib/ceph/mon/ceph-ceph-node2/done
	[ceph-node2][DEBUG ] create a done file to avoid re-doing the mon deployment
	[ceph-node2][DEBUG ] create the init path if it does not exist
	[ceph-node2][INFO  ] Running command: systemctl enable ceph.target
	[ceph-node2][INFO  ] Running command: systemctl enable ceph-mon@ceph-node2
	[ceph-node2][INFO  ] Running command: systemctl start ceph-mon@ceph-node2
	[ceph-node2][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.ceph-node2.asok mon_status
	[ceph-node2][WARNIN] ceph-node2 is not defined in `mon initial members`
	[ceph-node2][WARNIN] monitor ceph-node2 does not exist in monmap
	[ceph-node2][INFO  ] Running command: ceph --cluster=ceph --admin-daemon /var/run/ceph/ceph-mon.ceph-node2.asok mon_status
	[ceph-node2][DEBUG ] ********************************************************************************
	[ceph-node2][DEBUG ] status for monitor: mon.ceph-node2
	[ceph-node2][DEBUG ] {
	[ceph-node2][DEBUG ]   "election_epoch": 0, 
	[ceph-node2][DEBUG ]   "extra_probe_peers": [
	[ceph-node2][DEBUG ]     "192.168.1.101:6789/0"
	[ceph-node2][DEBUG ]   ], 
	[ceph-node2][DEBUG ]   "feature_map": {
	[ceph-node2][DEBUG ]     "mon": [
	[ceph-node2][DEBUG ]       {
	[ceph-node2][DEBUG ]         "features": "0x3ffddff8ffacfffb", 
	[ceph-node2][DEBUG ]         "num": 1, 
	[ceph-node2][DEBUG ]         "release": "luminous"
	[ceph-node2][DEBUG ]       }
	[ceph-node2][DEBUG ]     ]
	[ceph-node2][DEBUG ]   }, 
	[ceph-node2][DEBUG ]   "features": {
	[ceph-node2][DEBUG ]     "quorum_con": "0", 
	[ceph-node2][DEBUG ]     "quorum_mon": [], 
	[ceph-node2][DEBUG ]     "required_con": "0", 
	[ceph-node2][DEBUG ]     "required_mon": [
	[ceph-node2][DEBUG ]       "kraken", 
	[ceph-node2][DEBUG ]       "luminous", 
	[ceph-node2][DEBUG ]       "mimic", 
	[ceph-node2][DEBUG ]       "osdmap-prune"
	[ceph-node2][DEBUG ]     ]
	[ceph-node2][DEBUG ]   }, 
	[ceph-node2][DEBUG ]   "monmap": {
	[ceph-node2][DEBUG ]     "created": "2019-09-30 09:43:49.405219", 
	[ceph-node2][DEBUG ]     "epoch": 1, 
	[ceph-node2][DEBUG ]     "features": {
	[ceph-node2][DEBUG ]       "optional": [], 
	[ceph-node2][DEBUG ]       "persistent": [
	[ceph-node2][DEBUG ]         "kraken", 
	[ceph-node2][DEBUG ]         "luminous", 
	[ceph-node2][DEBUG ]         "mimic", 
	[ceph-node2][DEBUG ]         "osdmap-prune"
	[ceph-node2][DEBUG ]       ]
	[ceph-node2][DEBUG ]     }, 
	[ceph-node2][DEBUG ]     "fsid": "d77fa2a9-198e-4a9f-a106-8dbabe043e8d", 
	[ceph-node2][DEBUG ]     "modified": "2019-09-30 09:43:49.405219", 
	[ceph-node2][DEBUG ]     "mons": [
	[ceph-node2][DEBUG ]       {
	[ceph-node2][DEBUG ]         "addr": "192.168.1.101:6789/0", 
	[ceph-node2][DEBUG ]         "name": "ceph-node1", 
	[ceph-node2][DEBUG ]         "public_addr": "192.168.1.101:6789/0", 
	[ceph-node2][DEBUG ]         "rank": 0
	[ceph-node2][DEBUG ]       }
	[ceph-node2][DEBUG ]     ]
	[ceph-node2][DEBUG ]   }, 
	[ceph-node2][DEBUG ]   "name": "ceph-node2", 
	[ceph-node2][DEBUG ]   "outside_quorum": [], 
	[ceph-node2][DEBUG ]   "quorum": [], 
	[ceph-node2][DEBUG ]   "rank": -1, 
	[ceph-node2][DEBUG ]   "state": "probing", 
	[ceph-node2][DEBUG ]   "sync_provider": []
	[ceph-node2][DEBUG ] }
	[ceph-node2][DEBUG ] ********************************************************************************
	[ceph-node2][INFO  ] monitor: mon.ceph-node2 is currently at the state of probing

	最后，查看用下面的命令检查法定人数状态：
	[root@ceph-node1 ceph]# ceph quorum_status --format json-pretty

	{
	    "election_epoch": 12,
	    "quorum": [
	        0,
	        1
	    ],
	    "quorum_names": [
	        "ceph-node1",
	        "ceph-node2"
	    ],
	    "quorum_leader_name": "ceph-node1",
	    "monmap": {
	        "epoch": 2,
	        "fsid": "d77fa2a9-198e-4a9f-a106-8dbabe043e8d",
	        "modified": "2019-09-30 12:00:27.121540",
	        "created": "2019-09-30 09:43:49.405219",
	        "features": {
	            "persistent": [
	                "kraken",
	                "luminous",
	                "mimic",
	                "osdmap-prune"
	            ],
	            "optional": []
	        },
	        "mons": [
	            {
	                "rank": 0,
	                "name": "ceph-node1",
	                "addr": "192.168.1.101:6789/0",
	                "public_addr": "192.168.1.101:6789/0"
	            },
	            {
	                "rank": 1,
	                "name": "ceph-node2",
	                "addr": "192.168.1.130:6789/0",
	                "public_addr": "192.168.1.130:6789/0"
	            }
	        ]
	    }
	}

#### 部署成功后，可以再Ceph状态中检查新增的monitor
		[root@ceph-node1 ceph]# ceph status
	  cluster:
	    id:     d77fa2a9-198e-4a9f-a106-8dbabe043e8d
	    health: HEALTH_OK
	 
	  services:
	    mon: 3 daemons, quorum ceph-node1,ceph-node2,ceph-node3
	    mgr: ceph-node1(active)
	    osd: 3 osds: 3 up, 3 in
	 
	  data:
	    pools:   0 pools, 0 pgs
	    objects: 0  objects, 0 B
	    usage:   3.0 GiB used, 27 GiB / 30 GiB avail
	    pgs:     

可能会在新的monitor节点上遇到浴室中相关的简介公告消息。要消除这些警告，需要在新的monitor节点上设置NTP（Network Time Protocol，网络时间协议）
	[root@ceph-node3 ~]# yum install ntpd
	[root@ceph-node3 ~]# systemctl restart ntpd
	[root@ceph-node3 ~]# systemctl enable ntpd

### 添加Ceph OSD
至此，我们已经有了一个包含3个monitor节点的Ceph集群。现在我们将继续扩展集群，添加更过的OSD节点。要完成这个任务，我们在ceph-node1机器上执行以下命令（除非特别指明）。

我们使用前面相同的方法添加OSD。

	[root@ceph-node1 ceph]# ceph-deploy disk list ceph-node2 ceph-node3
	[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
	[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/bin/ceph-deploy disk list ceph-node2 ceph-node3
	[ceph_deploy.cli][INFO  ] ceph-deploy options:
	[ceph_deploy.cli][INFO  ]  username                      : None
	[ceph_deploy.cli][INFO  ]  verbose                       : False
	[ceph_deploy.cli][INFO  ]  debug                         : False
	[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
	[ceph_deploy.cli][INFO  ]  subcommand                    : list
	[ceph_deploy.cli][INFO  ]  quiet                         : False
	[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f83657e84d0>
	[ceph_deploy.cli][INFO  ]  cluster                       : ceph
	[ceph_deploy.cli][INFO  ]  host                          : ['ceph-node2', 'ceph-node3']
	[ceph_deploy.cli][INFO  ]  func                          : <function disk at 0x7f8365a31938>
	[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
	[ceph_deploy.cli][INFO  ]  default_release               : False
	[ceph-node2][DEBUG ] connected to host: ceph-node2 
	[ceph-node2][DEBUG ] detect platform information from remote host
	[ceph-node2][DEBUG ] detect machine type
	[ceph-node2][DEBUG ] find the location of an executable
	[ceph-node2][INFO  ] Running command: fdisk -l
	[ceph-node2][INFO  ] Disk /dev/sda: 21.5 GB, 21474836480 bytes, 41943040 sectors
	[ceph-node2][INFO  ] Disk /dev/sdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
	[ceph-node2][INFO  ] Disk /dev/sdc: 10.7 GB, 10737418240 bytes, 20971520 sectors
	[ceph-node2][INFO  ] Disk /dev/sdd: 10.7 GB, 10737418240 bytes, 20971520 sectors
	[ceph-node2][INFO  ] Disk /dev/mapper/centos-root: 18.2 GB, 18249416704 bytes, 35643392 sectors
	[ceph-node2][INFO  ] Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
	[ceph-node3][DEBUG ] connected to host: ceph-node3 
	[ceph-node3][DEBUG ] detect platform information from remote host
	[ceph-node3][DEBUG ] detect machine type
	[ceph-node3][DEBUG ] find the location of an executable
	[ceph-node3][INFO  ] Running command: fdisk -l
	[ceph-node3][INFO  ] Disk /dev/sda: 21.5 GB, 21474836480 bytes, 41943040 sectors
	[ceph-node3][INFO  ] Disk /dev/sdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
	[ceph-node3][INFO  ] Disk /dev/sdc: 10.7 GB, 10737418240 bytes, 20971520 sectors
	[ceph-node3][INFO  ] Disk /dev/sdd: 10.7 GB, 10737418240 bytes, 20971520 sectors
	[ceph-node3][INFO  ] Disk /dev/mapper/centos-root: 18.2 GB, 18249416704 bytes, 35643392 sectors
	[ceph-node3][INFO  ] Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
	
	# 选择在哪些磁盘上创建OSD
	[root@ceph-node1 ceph]# ceph-deploy disk zap ceph-node2 /dev/sdb  /dev/sdc /dev/sdd
	[root@ceph-node1 ceph]# ceph-deploy osd create ceph-node2 --data /dev/sdb
	[root@ceph-node1 ceph]# ceph-deploy osd create ceph-node2 --data /dev/sdc
	[root@ceph-node1 ceph]# ceph-deploy osd create ceph-node2 --data /dev/sdd


	[root@ceph-node1 ceph]# ceph-deploy disk zap ceph-node3 /dev/sdb /dev/sdc /dev/sdd
	[root@ceph-node1 ceph]# ceph-deploy osd create ceph-node3 --data /dev/sdb
	[root@ceph-node1 ceph]# ceph-deploy osd create ceph-node3 --data /dev/sdc
	[root@ceph-node1 ceph]# ceph-deploy osd create ceph-node3 --data /dev/sdd
		

	[root@ceph-node1 ceph]# ceph status
	# 检查集群中新增OSD的状态
	
	[root@ceph-node1 ceph]# ceph status
	  cluster:
	    id:     6fc37dfe-9613-4054-92ba-4cb60ce1bd98
	    health: HEALTH_WARN
	            clock skew detected on mon.ceph-node3, mon.ceph-node2
	 
	  services:
	    mon: 3 daemons, quorum ceph-node1,ceph-node3,ceph-node2
	    mgr: ceph-node1(active), standbys: ceph-node2, ceph-node3
	    osd: 9 osds: 9 up, 9 in
	 
	  data:
	    pools:   0 pools, 0 pgs
	    objects: 0  objects, 0 B
	    usage:   9.0 GiB used, 81 GiB / 90 GiB avail
	    pgs:  

## ceph 之mgr功能模块配置
### 添加mgr功能

	[root@ceph-node1 ~]# ceph-deploy mgr create ceph-node1 ceph-node2 ceph-node3

### 开启dashboard功能
	[root@ceph-node1 ~]# ceph mgr module enable dashboard
### 创建证书
	[root@ceph-node1 ~]# ceph dashboard create-self-signed-cert
### 创建web登录用户密码
	[root@ceph-node1 ~]# ceph dashboard set-login-credentials user-name password
### 查看服务访问方式
	[root@ceph-node1 ~]# ceph mgr services
	{
	    "dashboard": "https://ceph-node1:8443/"
	}

## 构建企业级分布式存储
### 硬件要求
一般选择2U的机型，磁盘STAT盘4T，如果I/O要求比较高，可以采购SSD固态硬盘。为了充分保证系统的稳定性和性能，要求所有存储服务器硬件配置尽量一致，尤其是硬盘数量和大小。
### 系统要求和分区划分
系统要求使用CentOS7.x，安装完成后升级到最新版本，安装的系统独立安装在系统盘（可做raid），建议/boot分区1G，/分区100G、swap分区和内存一样大小，剩余磁盘给ceph使用。系统安装软件没有特殊要求，建议除了开发工具和基本管理软件，其他软件一律不安装。

### 网络环境
网络要求全部千兆环境，ceph服务至少有2块网卡，1块网卡绑定供ceph使用，剩余一块网卡分配管理网络IP，用户系统管理。如果有条件购买万兆交换机，服务器配置万兆网卡，存储性能会更好。网络方面如果安全性要求较高，可以多网卡绑定

### 服务器摆放分布
服务器主备机器要放在不同的机柜，连接不通的交换机，即使一个机柜出现问题，还有一份数据正常访问。

### 构建高性能、高可用存储
一般在企业中，采用的3+副本,保证数据可用性.

#### 开放防火墙端口
一般在企业应用中Linux防火墙是打开的，这些开通服务器之间访问的端口：6789 8443 6701-6800等
		
## 总结
Ceph 的软件定义性质为其使用者提供了极大的灵活性。不像其他专有存储都依赖于硬件，Ceph几乎可以为任何计算机系统轻松进行部署和测试。此外。如果没有物理机器，你可以使用虚拟机安装Ceph，正如前面所述的，但这只用于学习和测试的目的

使用ceph-deploy工具部署一个三节点的Ceph集群。我们也增加了几个OSD和monitor节点到集群中，以展示其动态的可伸缩性。
