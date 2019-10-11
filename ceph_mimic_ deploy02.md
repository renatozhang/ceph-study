
# Ceph集群手工部署

## 安装依赖
Ceph需要安装第三方库。对于RHEL系列的linux发行版本，可以从EPEL存储库获取依赖包。执行以下命令安装Ceph
### 安装epel库
	[root@ceph-node1 ~]# yum install epel-release -y
### 安装Ceph需要的第三方二进制文件
	[root@ceph-node1 ~]# yum install -y snappy leveldb gdisk python-argparse gperftools-libs
### 创建一个Ceph存储库文件用于描述Mimic发行版
	[root@ceph-node1 ~]# cat /etc/yum.repos.d/ceph.repo 
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
	
### 安装Ceph软件包
	[root@ceph-node1 ~]# yum install -y ceph
### 验证是否安装成功
	[root@ceph-node1 ~]# rpm -qa | egrep -i "ceph|rados|rbd"
	librados2-13.2.6-0.el7.x86_64
	libradosstriper1-13.2.6-0.el7.x86_64
	ceph-base-13.2.6-0.el7.x86_64
	ceph-mon-13.2.6-0.el7.x86_64
	librbd1-13.2.6-0.el7.x86_64
	libcephfs2-13.2.6-0.el7.x86_64
	python-rbd-13.2.6-0.el7.x86_64
	ceph-common-13.2.6-0.el7.x86_64
	ceph-selinux-13.2.6-0.el7.x86_64
	ceph-mds-13.2.6-0.el7.x86_64
	ceph-osd-13.2.6-0.el7.x86_64
	python-rados-13.2.6-0.el7.x86_64
	python-cephfs-13.2.6-0.el7.x86_64
	ceph-mgr-13.2.6-0.el7.x86_64
	ceph-13.2.6-0.el7.x86_64
### 在其他三个Ceph节点ceph-node2、ceph-node3、ceph-node4上执行以上步骤

## 部署Ceph集群
现在已经在所有节点上安装好Ceph，并且可以开始Ceph的手工部署了。手工部署过程主要由哪些配置管理工具开发部署脚本的人员使用，比如ansible、chef以及puppet，需要知道部署流程。作为一名管理员，可以使用手工部署的方法来增加Ceph经验。大部分生产环境的部署基于ceph-deploy工具。



## NTP服务配置
	[root@ceph-node2 ~]# vim /etc/ntp.conf
	server 192.168.1.101  iburst
	### 手动同步下时间
	[root@ceph-node2 ~]# ntpdate -u 192.168.1.101 
	### 启动服务
	[root@ceph-node2 ~]# systemctl restart ntpd
	[root@ceph-node2 ~]# systemctl enable ntpd 
	### 检查同步
	[root@ceph-node2 ~]# ntpq -p
## 部署monitor
### 为Ceph创建一个目录并且创建Ceph集群文件
	[root@ceph-node1 ceph]# mkdir /etc/ceph
	[root@ceph-node1 ceph]# touch /etc/ceph/ceph.conf

### 为集群生成一个fsid
	[root@ceph-node1 ceph]# uuidgen 
	d30700a1-e1a7-407b-aacd-0b82ed71fee7
### 创建集群配置文件。
默认集群名字为ceph。配置文件名称是/etc/ceph/ceph.conf

使用前面步骤中uuidgen命令输出作为配置文件中的fsid参数

	[root@ceph-node1 ~]# cat /etc/ceph/ceph.conf
	[global]
	fsid = d30700a1-e1a7-407b-aacd-0b82ed71fee7
	
	[mon.ceph-node1]
	host = ceph-node1
	mon addr = 192.168.1.101:6789

### 为集群创建秘钥环，并使用如下命令生成monitor的秘钥
	[root@ceph-node1 ceph]#  ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
	creating /tmp/ceph.mon.keyring
### 创建管理员秘钥环 生成client.admin用户并将其添加到秘钥环上
	[root@ceph-node1 ceph]#  ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
	creating /etc/ceph/ceph.client.admin.keyring
### 创建一个bootstrp-osd秘钥环，生成client.bootstrap-osd用户，然后添加用户环上
	[root@ceph-node1 ceph]#  ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd'
	creating /var/lib/ceph/bootstrap-osd/ceph.keyring

### 将client.admin秘钥导入到ceph.mon.keyring中
	[root@ceph-node1 ceph]#  ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
	importing contents of /etc/ceph/ceph.client.admin.keyring into /tmp/ceph.mon.keyring

	[root@ceph-node1 ceph]#  ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
	importing contents of /var/lib/ceph/bootstrap-osd/ceph.keyring into /tmp/ceph.mon.keyring

### 为第一个monitor生成monitor map 创建monitor映射
monmaptool --create --add {hostname} {ip-address} --fsid 45ef193b-3d8f-496f-8a96-cba540001c3b /tmp/monmap

	[root@ceph-node1 ceph]# monmaptool --create --add ceph-node1 192.168.1.101 --fsid d30700a1-e1a7-407b-aacd-0b82ed71fee7 /tmp/monmap
	monmaptool: monmap file /tmp/monmap
	monmaptool: set fsid to 9fc2c5e4-7417-4f22-ad57-b0cab6187c84
	monmaptool: writing epoch 0 to /tmp/monmap (1 monitors)
		
### 为monitor创建类似/path/cluster_name-monitor_node格式的目录作为默认数据目录
	[root@ceph-node1 ~]# mkdir /var/lib/ceph/mon/ceph-ceph-node1
### 添加第一个monitor守护进程信息 使monitor映射和秘钥环填充monitor服务
	[root@ceph-node1 ~]# ceph-mon --mkfs -i ceph-node1 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring  
### 标记监视器服务就绪
	[root@ceph-node1 ceph]# touch /var/lib/ceph/mon/ceph-ceph-node1/done

### 更新配置文件
	[root@ceph-node1 ~]# cat /etc/ceph/ceph.conf 
	[global]
	fsid = d30700a1-e1a7-407b-aacd-0b82ed71fee7
	auth cluster required = cephx
	auth service required = cephx
	auth client required = cephx
	osd journal size = 1024
	osd pool default size = 2
	osd pool default min size = 1
	osd pool default pg num = 128
	osd pool default pgp num = 128
	osd crush chooseleaf type = 1
	
	[mon.ceph-node1]
	host = ceph-node1
	mon addr = 192.168.1.101:6789
### 启动monitor服务
1. 修改数据目录权限

		chown -R ceph:ceph /var/lib/ceph	

2. 修改服务启动脚本
	
		[root@ceph-node1 ceph]# vim /usr/lib/systemd/system/ceph-mon@.service 
		ExecStart=/usr/bin/ceph-mon -f --cluster ${CLUSTER} --id %i --setuser root --setgroup root
3. 启动服务

		[root@ceph-node1 ~]# systemctl daemon-reload
		[root@ceph-node1 ~]# systemctl start ceph-mon@ceph-node1
	    [root@ceph-node1 ~]# systemctl enable ceph-mon@ceph-node1
	    [root@ceph-node1 ~]# ceph -s
		  cluster:
		    id:     d30700a1-e1a7-407b-aacd-0b82ed71fee7
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
### 添加mon服务
#### 拷贝配置文件和客户端秘钥
	[root@ceph-node3 ~]# scp root@192.168.1.101:/etc/ceph/ceph.conf /etc/ceph/
	[root@ceph-node3 ~]# scp root@192.168.1.101:/etc/ceph/ceph.client.admin.keyring /etc/ceph/
#### 修改配置文件
	[root@ceph-node3 ~]# cat /etc/ceph/ceph.conf 
	[global]
	fsid = 38f45748-f634-48dd-a196-8593562cd385
	auth cluster required = cephx
	auth service required = cephx
	auth client required = cephx
	osd journal size = 1024
	osd pool default size = 2
	osd pool default min size = 1
	osd pool default pg num = 128
	osd pool default pgp num = 128
	osd crush chooseleaf type = 1
	
	[mon.ceph-node3]
	host = ceph-node3
	mon addr = 192.168.1.103:6789
	# 配置完成前需要保留下面的配置
	[mon.ceph-node1]
	host = ceph-node1
	mon addr = 192.168.1.101:6789

#### 获取mon集群秘钥环
	[root@ceph-node3 ~]# ceph auth get mon. -o /tmp/ceph.mon.keyring
#### 获取mon映射文件（monmap）
	[root@ceph-node3 ~]# ceph mon getmap -o /tmp/monmap
#### 使用mon映射和秘钥环境填充新mon服务
	[root@ceph-node3 ~]# ceph-mon --mkfs -i ceph-node3 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring 
#### 标记mon服务就绪
	[root@ceph-node3 ~]# touch /var/lib/ceph/mon/ceph-ceph-node3/done
#### 启动mon服务
	[root@ceph-node3 ~]# chown -R ceph:ceph /var/lib/ceph/
	[root@ceph-node3 ~]# systemctl start ceph-mon@ceph-node3
	[root@ceph-node3 ~]# systemctl enable ceph-mon@ceph-nodde3

#### 检测服务
	[root@ceph-node3 ~]# ceph -s
	  cluster:
	    id:     38f45748-f634-48dd-a196-8593562cd385
	    health: HEALTH_OK
	 
	  services:
	    mon: 3 daemons, quorum ceph-node1,ceph-node2,ceph-node3
	    mgr: no daemons active
	    osd: 0 osds: 0 up, 0 in
	 
	  data:
	    pools:   0 pools, 0 pgs
	    objects: 0  objects, 0 B
	    usage:   0 B used, 0 B / 0 B avail
	    pgs:     	 
## 配置管理服务
### 创建密钥文件
	[root@ceph-node1 ~]# mkdir -p /var/lib/ceph/mgr/ceph-ceph-node1
	[root@ceph-node1 ~]# ceph auth get-or-create mgr.ceph-node1 mon "allow profile mgr" osd "allow *" mds 'allow *' -o /var/lib/ceph/mgr/ceph-ceph-node1/keyring
### 启动服务
	[root@ceph-node1 ~]# systemctl start ceph-mgr@ceph-ceph-node1
	[root@ceph-node1 ~]# systemctl enable ceph-mgr@ceph-ceph-node1
### 查询服务
	[root@ceph-node1 ~]# ceph -s
	  cluster:
	    id:     38f45748-f634-48dd-a196-8593562cd385
	    health: HEALTH_OK
	 
	  services:
	    mon: 3 daemons, quorum ceph-node1,ceph-node2,ceph-node3
	    mgr: ceph-node1(active), standbys: ceph-node2, ceph-node3
	    osd: 1 osds: 1 up, 1 in
	 
	  data:
	    pools:   0 pools, 0 pgs
	    objects: 0  objects, 0 B
	    usage:   1.0 GiB used, 9.0 GiB / 10 GiB avail
	    pgs:     
 
### 同理配置好node2和node3节点


## 监控节点配置（方式二）
> 在ceph-node1上执行
### 为CEPH集群生成UUID
	# uuidgen
	cb9321ef-c7b4-48f7-a1bf-5c75deede6ee
	# tee /etc/ceph/ceph.conf << EOF
	[global]
	fsid = cb9321ef-c7b4-48f7-a1bf-5c75deede6ee
	auth cluster required = cephx
	auth service required = cephx
	auth client required = cephx
	osd journal size = 1024
	osd pool default size = 2
	osd pool default min size = 1
	osd pool default pg num = 128
	osd pool default pgp num = 128
	osd crush chooseleaf type = 1
	mon initial members = ceph-node1, ceph-node2, ceph-node3
	mon host = 192.168.1.101, 192.168.1.102, 192.168.1.103
	EOF

### 创建集群密钥环以及监视器密钥
		# ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
### 创建管理员密钥环，生成client.admin用户，然后添加用户到环上
		# ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
### 创建一个bootstrap-osd密钥环，生成client.bootstrap-osd用户，然后添加用户到环上
		# ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd'
### 将生成的密钥添加到ceph.mon.keyring
		# ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
		# ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
### 创建监视器映射
		# monmaptool --create --add ceph-node1 192.168.1.101 --add ceph-node2 192.168.1.102 --add ceph-node3 192.168.1.103 --fsid cb9321ef-c7b4-48f7-a1bf-5c75deede6ee --clobber /tmp/monmap
		# monmaptool --print /tmp/monmap
### 创建默认数据目录
		# mkdir -p /var/lib/ceph/mon/ceph-ceph-node1
### 使用监视器映射和密钥环填充监视器服务
		# ceph-mon --mkfs -i ceph-node1 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
### 标记监视器服务就绪
	# touch /var/lib/ceph/mon/ceph-anode/done
### 启动监视器服务
	### 启动服务
	# chown -R ceph:ceph /var/lib/ceph
	# systemctl start ceph-mon@ceph-node2
	# systemctl enable ceph-mon@anode
	### 检测服务
	# ceph daemon mon.anode mon_status
### 添加监控节点
> 以下指令在ceph-node2节点执行

### 拷贝配置文件、密钥和监视器映射
	# scp root@192.168.1.101:/etc/ceph/ceph.conf /etc/ceph/
	# scp root@192.168.1.101:/tmp/ceph.mon.keyring /tmp
	# scp root@192.168.1.101:/tmp/monmap /tmp
### 创建默认数据目录
	# mkdir -p /var/lib/ceph/mon/ceph-bnode
### 使用监视器映射和密钥环填充监视器服务
	# ceph-mon --mkfs -i ceph-node2 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
### 标记监视器服务就绪
	# touch /var/lib/ceph/mon/ceph-ceph-node2/done
### 启动监视器服务
	### 启动服务
	# chown -R ceph:ceph /var/lib/ceph
	# systemctl start ceph-mon@ceph-node2
	# systemctl enable ceph-mon@ceph-node2
	### 检测服务
	# ceph daemon mon.ceph-node2 mon_status
### 同理添加ceph-node3节点
			

## 创建OSD
### 脚本创建（ceph-node1）
	[root@ceph-node1 ~]#  ceph-volume lvm create --data /dev/sdb
### 手动（ceph-node1)
#### 拷贝配置文件和秘钥环到部署OSD的节点（此处为mon也作为osd无需再拷贝）

	# scp root@192.168.1.101:/etc/ceph/ceph.conf /etc/ceph/
	# scp root@192.168.1.101:/etc/ceph/ceph.client.admin.keyring /etc/ceph/
	# scp root@192.168.1.101:/var/lib/ceph/bootstrap-osd/ceph.keyring /var/lib/ceph/bootstrap-osd/
#### 为OSD生成UUID
	[root@ceph-node1 ~]# UUID=$(uuidgen)
#### 为OSD创建cephx秘钥
	[root@ceph-node1 ~]# OSD_SECRET=$(ceph-authtool --gen-print-key)
#### 创建OSD
	### ceph命令-i原本是要指定一个json文件，这边使用管道操作中的"-"将前面echo的输出结果作为文件
	[root@ceph-node1 ~]# ID=$(echo "{\"cephx_secret\": \"$OSD_SECRET\"}" | ceph osd new $UUID -i - -n client.bootstrap-osd -k /var/lib/ceph/bootstrap-osd/ceph.keyring)
#### 为OSD创建数据目录
	[root@ceph-node1 ~]# mkdir /var/lib/ceph/osd/ceph-$ID
#### 挂在磁盘导数据目录
	[root@ceph-node1 ~]# mkfs.xfs /dev/sdc
	[root@ceph-node1 ~]# mount /dev/sdc /var/lib/ceph/osd/ceph-$ID
#### 创建OSD秘钥文件
	
#### 修改fstab配置
	### 在配置文件末尾添加
	[root@ceph-node1 ~]# vim /etc/fstab 
	/dev/sdc /var/lib/ceph/osd/ceph-1 xfs defaults 0 0
#### 创建OSD秘钥文件
	[root@ceph-node1 ~]# ceph-authtool --create-keyring /var/lib/ceph/osd/ceph-$ID/keyring --name osd.$ID --add-key $OSD_SECRET
#### 初始化OSD数据目录
	[root@ceph-node1 ~]# ceph-osd -i $ID --mkfs --osd-uuid $UUID
#### 启动服务

	[root@ceph-node1 ~]# chown -R ceph:ceph /var/lib/ceph/osd/ceph-$ID
	[root@ceph-node1 ~]# systemctl enable ceph-osd@$ID
	[root@ceph-node1 ~]# systemctl start ceph-osd@$ID 
	[root@ceph-node1 ceph-0]# ceph -s
	  cluster:
	    id:     9e888166-737a-41de-8f07-ada88d471314
	    health: HEALTH_OK
	 
	  services:
	    mon: 3 daemons, quorum ceph-node1,ceph-node2,ceph-node3
	    mgr: ceph-node1(active), standbys: ceph-node2, ceph-node3
	    osd: 9 osds: 9 up, 9 in
	 
	  data:
	    pools:   0 pools, 0 pgs
	    objects: 0  objects, 0 B
	    usage:   46 GiB used, 44 GiB / 90 GiB avail
	    pgs:     



# 升级Ceph集群
