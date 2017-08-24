---
layout: post
title: "NetApp 系统概念和基本命令行操作"
---

```
$ ssh admin@$address
# 查看有哪些aggregate
redhat::> version 
NetApp Release 8.2P3 Cluster-Mode: Thu Sep 07 00:01:01 PDT 2018
redhat::> storage aggregate show -has-mroot true  # -has-mroot 不懂什么意思??
Aggregate     Size Available Used% State   #Vols  Nodes            RAID Status
--------- -------- --------- ----- ------- ------ ---------------- ------------
aggr_1      1.92TB    1.33TB   31% online       5 redhat-01        raid_dp,
                                                                   normal
aggr_2      1.92TB    1.87TB    3% online       1 redhat-02        raid_dp,
                                                                   normal
aggr_3     14.55TB   14.55TB    0% online       0 redhat-01        raid_dp,
                                                                   normal
3 entries were displayed.

redhat::> vserver create -vserver nfs-mds -aggregate aggr_3 -rootvolume root -ns-switch file -rootvolume-security-style mixed -language en_US
redhat::> vserver setup -vserver nfs-mds -storage true -network true -protocols nfs,cifs
...
redhat::> vserver nfs modify -vserver nfs-mds -access true -tcp enabled -v3 enabled -v4.0 enabled -V4.0-read-delegation enabled -V4.0-write-delegation enabled -V4.1 enabled -V4.1-pnfs enabled  -V4.1-read-delegation enabled -V4.1-write-delegation enabled
redhat::> vserver nfs modify -vserver nfs-ds -access true -tcp enabled -v3 enabled -v4.0 enabled -V4.0-read-delegation enabled -V4.0-write-delegation enabled -V4.1 enabled -V4.1-pnfs enabled  -V4.1-read-delegation enabled -V4.1-write-delegation enabled
```

# Aggregate, Vserver, Node, Volume, Junction

## Aggregate (*just like VG in LVM)
http://www.netpro.com.tw/?FID=23&CID=147&category=2

```
#storage aggregate show -has-mroot false

#vserver create -vserver vserver_name -aggregate aggregate_name -rootvolume root_volume_name \
 -rootvolume-security-style {unix|ntfs|mixed} -ns-switch {nis|file|ldap},... \
 [-nm-switch {file|ldap},...] [-language language] [-snapshot-policy snapshot_policy_name] \
 [-quota-policy quota_policy_name] -comment comment]

cluster1::> vserver create -vserver vs1 -aggregate aggr3 -rootvolume vs1_root -ns-switch file \
 -rootvolume-security-style ntfs -language en_US
 [Job 72] Job succeeded:                 Vserver creation completed
cluster1::> vserver show -vserver vs1
				   Vserver: vs1
			      Vserver Type: data
			      Vserver UUID: 11111111-1111-1111-1111-111111111111
			       Root Volume: vs1_root
				 Aggregate: aggr3
		       Name Service Switch: file
		       Name Mapping Switch: file
				NIS Domain: -
		Root Volume Security Style: ntfs
			       LDAP Client: -
				  Language: en_US
			   Snapshot Policy: default
				   Comment:
		Antivirus On-Access Policy: default
			      Quota Policy: default
	       List of Aggregates Assigned: -
Limit on Maximum Number of Volumes allowed: unlimited
		       Vserver Admin State: running
			 Allowed Protocols: nfs, cifs, ndmp
		      Disallowed Protocols: fcp, iscsi
	   Is Vserver with Infinite Volume: false
			  QoS Policy Group: 
```

```
redhat::> aggr show
Aggregate     Size Available Used% State   #Vols  Nodes            RAID Status
--------- -------- --------- ----- ------- ------ ---------------- ------------
aggr0      492.2GB   22.90GB   95% online       1 redhat-02        raid_dp,
                                                                   normal
aggr0_redhat_02_0 
           492.2GB   22.90GB   95% online       1 redhat-02        raid_dp,
                                                                   normal
aggr_1      1.92TB    1.36TB   29% online       6 redhat-02        raid_dp,
                                                                   normal
aggr_2      1.92TB    1.81TB    6% online       2 redhat-02        raid_dp,
                                                                   normal
aggr_3     14.55TB   14.54TB    0% online       4 redhat-02        raid_dp,
                                                                   normal
5 entries were displayed.

redhat::> storage aggregate show
Aggregate     Size Available Used% State   #Vols  Nodes            RAID Status
--------- -------- --------- ----- ------- ------ ---------------- ------------
aggr0      492.2GB   22.90GB   95% online       1 redhat-02        raid_dp,
                                                                   normal
aggr0_redhat_02_0 
           492.2GB   22.90GB   95% online       1 redhat-02        raid_dp,
                                                                   normal
aggr_1      1.92TB    1.36TB   29% online       6 redhat-02        raid_dp,
                                                                   normal
aggr_2      1.92TB    1.81TB    6% online       2 redhat-02        raid_dp,
                                                                   normal
aggr_3     14.55TB   14.54TB    0% online       4 redhat-02        raid_dp,
                                                                   normal
5 entries were displayed.
```

## Vserver, Node

现在叫 VSM(Virtual Storage Machine), 拥有独立的文件系统和网络命名空间，可以理解为一个容器
vserver 三种类型 data # 还有 admin node, 这两种类型没有 root-volume, 而且是自动创建的
The cluster setup process automatically creates the admin Vserver for the cluster. 
A node Vserver is created when the node joins the cluster.

```
	vs1::> vserver show -vserver vs1 -fields allowed-protocols
	vserver allowed-protocols
	------- -----------------
	vs1     nfs
	vs1::> vserver modify -vserver vs1 -allowed-protocols nfs,cifs

	# vserver nfs modify -vserver vserver_name -v3 enabled
	# vserver nfs modify -vserver vserver_name -v3 disabled
	# vserver nfs modify -vserver vserver_name -v4.0 enabled
	# vserver nfs modify -vserver vserver_name -v4.0 disabled
	# vserver nfs modify -vserver vserver_name -v4.1 enabled
	# vserver nfs modify -vserver vserver_name -v4.1 disabled
	# vserver nfs modify -vserver vserver_name -v4.1-pnfs enabled
	# vserver nfs modify -vserver vserver_name -v4.1-pnfs disabled
	# #-v4.0-acl -v4.1-acl -v4-acl-preserv  -v4.0-read-delegation -v4.1-read-delegation
	# #-v4.0-write-delegation -v4.1-write-delegation ...
	# #more details in {Managing file access using NFS}

	vs1::> vserver nfs create -vserver vs1 -v3 disabled -v4.0 enabled -v4.0-acl enabled
```

Creating an export policy
Adding a rule to an export polic

```
	# vserver export-policy create -vserver virtual_server_name -policyname policy_name
	# vserver export-policy rule create -vserver virtual_server_name -policyname policy_name \
	 -ruleindex integer -protocol {any|nfs3|nfs|cifs|nfs4|flexcache},... -clientmatch text \
	 -rorule {any|none|never|krb5|ntlm|sys},... -rwrule {any|none|never|krb5|ntlm|sys},... \
	 -anon user_ID -superuser {any|none|krb5|ntlm|sys},... -allow-suid {true|false} \
	 -allow-dev {true|false}
	# vserver export-policy rule setindex -vserver virtual_server_name -policyname policy_name \
	 -ruleindex integer -newruleindex integer

	vs1::> vserver export-policy rule setindex -vserver vs1 -policyname rs1 -ruleindex 3 -newruleindex 2

	cluster::> volume create -vserver vs1 -volume vol1 \
	-aggregate aggr2 -state online -policy cifs_policy -security-style ntfs \
	-junction-path /dept/marketing -size 250g -space-guarantee volum

	cluster::> vserver export-policy rule show -policyname cifs_policy
	Virtual      Policy             Rule    Access   Client       RO
	Server       Name               Index   Protocol Match        Rule
	------------ ------------------ ------  -------- ------------ ------
	vs1          cifs_policy        1       cifs      0.0.0.0/0    a

	cluster::> vserver export-policy rule show -policyname cifs_policy -vserver vs1 -ruleindex 1
				     Virtual Server: vs1 
					Policy Name: cifs_policy 
					 Rule Index: 1 
				    Access Protocol: cifs
				  Client Match Spec: 0.0.0.0/0 
				     RO Access Rule: any 
				     RW Access Rule: any 
	User ID To Which Anonymous Users Are Mapped: 0 
			 Superuser Security Flavors: any 
		       Honor SetUID Bits In SETATTR: true 
			  Allow Creation of Devices: true
```

## Volume
可以理解为逻辑卷,类似我们通常说的文件系统分区
{FlexVol volume} {Infinite Volume} 区别:
#https://library.netapp.com/ecmdocs/ECMP1196906/html/GUID-58EC88D8-29F9-4865-88B9-A4FB66224FEE.html
SVM里面的FlexVol可以支持文件或者LUN，大家都知道，NETAPP的LUN是在文件系统WAFL上仿真出来的。SVM也支持
infinite volume，这种卷可以不指定空间，随便扩展（最大20PB)，但只支持文件系统，不支持块设备

```
Flexible volume建立在Vserver(VSM)上面，是clustered Data ONTAP namespace的基础。

一个volume是aggregate的一部分，在volume上面包含文件和目录。不需要增加或是减少物理资源就可以动态的
调整volume的尺寸。Data ONTAP 7-mode系统有两个类型的volume，一个是traditional，一个是
flexible volume。ClusteredData ONTAP只用一种类型的volume，就是Flexible volume。在这两种系统中
的Flexible volume是相同的。Cluster使用volume来对数据进行管理。通常是以volume为单位来进行快照拷贝、
镜像和磁带备份的。Volume可以在一个cluster内部进行移动。并且volume的移动对于正在使用这些volume的
用户来说是透明的。

Clustered Data ONTAP的一个关键特性是可以将volume组合在一起，建立一个分布式的全局namespaces。如果
一个客户端挂载或是映射一个namespace的rootvolume，他就能够导航到这个namespace上的每一个volume，而
不必关心这个volume位于这个cluster上的那个node上面。

将所有的volume集合在一起就被称为是一个junction。从概念上来看，一个junction就象是UNIX系统中的一个
mountpoint。UNIX系统的mount point是将一个UNIX分区连接的另一个分区，junction将一个volume连接到另
一个volume。从一个客户机角度看，junction简单看上去就像一个目录。Namespace中的一个volume可以连接到
namespace中的其他的每一个volume。在父volume内部，一个junction可以位于任意一个目录或是qtree。一个
volume能够从他的父volume解除mount，然后mount到另一个父volume上。

在建立vserver的时候需要指定使用哪个aggregate，也就是说vserver是建立在aggregate之上的。
在建立vserver的时候也要输入一个域名。

课程的module5的第八页有建立vserver的操作练习，是通过system manager进行的。

Clustered DataONTAP 8.1.1系统引入了infinitevolume。Infinite volume是一个无边界的、易管理的和可
扩展的容器，infinite volume超出了当前的DataONTAP的flexible volume的限制。Clustered Data ONTAP 8.2
改善了infinitevolume。Infinite volume可以支持NFSv3、CIFS、NFSv4.1和pNFS。DataONTAP 8.2引入了
data constituent group的概念，能够用于实现存储级的数据分层。

Infinite volume能够和Flexiblevolume在aggregate上同时存在，也就是说可以和flexible volume共享一个
aggregate。启动了infinitevolume的vserver和支持flexible volume的vserver也可以同时存在。
Infinite volume采用一个统一的安全设计，统一的安全设置可以提供统一的访问控制列表，这个访问列表可以幫
助进行访问检查。Infinite volume可以与另一个cluster上的infinitevolume建立双向的镜像关系或是单向输出
关系，利用者个功能，Clustered Data ONTAP 8.2可以从镜像的namespace中恢复本地的永久丢失的namespace中
的数据。FAS2000系列不支持infinite volume。

建立infinitevolume和承载infinite volume的aggregate的方式与建立FlexVol volume的方式是类似的。

Volume是不依赖于node的，volume是和vserver相关联的。在clustered-mode环境中，不能建立traditionalvolume。
```

Creating data volumes with specified junction point
```
	# volume create -vserver vserver_name -volume volume_name -aggregate aggregate_name \
	 -size {integer[KB|MB|GB|TB|PB]} -security-style {ntfs|unix|mixed} -junction-path junction_path
	# volume show -vserver vserver_name -volume volume_name -junction

	cluster1::> volume create -vserver vs1 -volume home4 -aggregate aggr1 -size 1g -junction-path /eng/home
	[Job 1642] Job succeeded: Successful

	cluster1::> volume show -vserver vs1 -volume home4 -junction
			  Junction                 Junction
	Vserver   Volume  Active   Junction Path   Path Source
	--------- ------- -------- --------------- -----------
	vs1       home4   true     /eng/home       RW_volume
```

Creating data volumes without specifying junction points

```
	# volume create -vserver vserver_name -volume volume_name -aggregate aggregate_name \
	 -size {integer[KB|MB|GB|TB|PB]} -security-style {ntfs|unix|mixed}
	# volume show -vserver vserver_name -volume volume_name -junction

	cluster1::> volume create -vserver vs1 -volume sales -aggregate aggr3 -size 20GB
	[Job 3406] Job succeeded: Successful

	cluster1::> volume show -vserver vs1 -junction
			     Junction                 Junction
	Vserver   Volume     Active   Junction Path   Path Source
	--------- ---------- -------- --------------- -----------
	vs1       data       true     /data           RW_volume
	vs1       home4      true     /eng/home       RW_volume
	vs1       vs1_root   -        /               -
	vs1       sales      -        -               -
```

Mounting or unmounting existing volumes in the NAS namespace
```
	# volume mount -vserver vserver_name -volume volume_name -junction-path junction_path
	# volume unmount -vserver vserver_name -volume volume_name
	# volume show -vserver vserver_name -volume volume_name -junction

	cluster1::> volume mount -vserver vs1 -volume sales -junction-path /sales
	cluster1::> volume show -vserver vs1 -junction
			     Junction                 Junction
	Vserver   Volume     Active   Junction Path   Path Source
	--------- ---------- -------- --------------- -----------
	vs1       data       true     /data           RW_volume
	vs1       home4      true     /eng/home       RW_volume
	vs1       vs1_root   -        /               -
	vs1       sales      true     /sales          RW_volume

	cluster1::> volume unmount -vserver vs1 -volume data
	cluster1::> volume show -vserver vs1 -junction
			     Junction                 Junction
	Vserver   Volume     Active   Junction Path   Path Source
	--------- ---------- -------- --------------- -----------
	vs1       data       -        -               -
	vs1       home4      true     /eng/home       RW_volume
	vs1       vs1_root   -        /               -
	vs1       sales      true     /sales          RW_volume
```

Displaying volume mount and junction point information
```
	cluster1::> volume show -vserver vs1 -junction
			     Junction                 Junction
	Vserver   Volume     Active   Junction Path   Path Source
	--------- ---------- -------- --------------- -----------
	vs1       data       true     /data           RW_volume
	vs1       home4      true     /eng/home       RW_volume
	vs1       vs1_root   -        /               -
	vs1       sales      true     /sales          RW_volume

	cluster1::> volume show -vserver vs2 -fields vserver,volume,aggregate,size,state,type,security-
	style,junction-path,junction-parent,node
	vserver volume   aggregate size state  type security-style junction-path junction-parent node
	------- ------   --------- ---- ------ ---- -------------- ------------- --------------- ----- 
	vs2     data1    aggr3     2GB  online RW   unix           -             -               node3
	vs2     data2    aggr3     1GB  online RW   ntfs           /data2        vs2_root        node3 
	vs2     data2_1  aggr3     8GB  online RW   ntfs           /data2/d2_1   data2           node3
	vs2     data2_2  aggr3     8GB  online RW   ntfs           /data2/d2_2   data2           node3
	vs2     pubs     aggr1     1GB  online RW   unix           /publications vs2_root        node1
	vs2     images   aggr3     2TB  online RW   ntfs           /images       vs2_root        node3
	vs2     logs     aggr1     1GB  online RW   unix           /logs         vs2_root        node1
	vs2     vs2_root aggr3     1GB  online RW   ntfs           /             -               node3
```

## Junction
由一个或多个已挂载的Volume组成的文件系统树.
```
redhat::> volume show -vserver vservername  -junction
		     Junction                       Junction
Vserver Volume       Active   Junction Path         Path Source
------- ------------ -------- -------------------   -----------
vs1     audit        true     /audit                RW_volume
vs1     audit_logs1  true     /audit/logs1          RW_volume
vs1     audit_logs2  true     /audit/logs2          RW_volume
vs1     audit_logs3  true     /audit/logs3          RW_volume
vs1     eng          true     /data/eng             RW_volume
vs1     mktg1        true     /data/mktg1           RW_volume
vs1     mktg2        true     /data/mktg2           RW_volume
vs1     project1     true     /projects/project1    RW_volume
vs1     project2     true     /projects/project2    RW_volume
vs1     vs1_root     -        /                     -        
```

## Security styles
Each volume and qtree on the storage system has a security style. 
The security style determines what type of permissions are used for data on volumes when authorizing user.
```
      	+------------------------------------------------------------------------------------+
      	| Security  | Client that | Permission that | Resulting effective | Clients that can |
        | style     | can modify  | clients can use | security style      | access files     |
        |           | permission  |                 |                     |                  |
        |-----------|-------------|-----------------|---------------------|------------------|
        | UNIX      | NFS         | NFSv3 mode bits | UNIX                | NFS and CIFS     |
        |           |             |-----------------|---------------------|                  |
        |           |             | NFSv4.x ACLs    | UNIX                |                  |
        |-----------|-------------|-----------------|---------------------|                  |
        | NTFS      | CIFS        | NTFS ACLs       | NTFS                |                  |
        |-----------|-------------|-----------------|---------------------|                  |
        | Mixed     | NFS or CIFS | NFSv3 mode bits | UNIX                |                  |
        |           |             |-----------------|---------------------|                  |
        |           |             | NFSv4.x ACLs    | UNIX                |                  |
        |           |             |-----------------|---------------------|                  |
        |           |             | NTFS ACLs       | NTFS                |                  |
        |-----------|-------------|-----------------|---------------------|                  |
        | Unified   | NFS or CIFS | NFSv3 mode bits | UNIX                |                  |
        | (only for |             |-----------------|---------------------|                  |
        | Infinite  |             | NFSv4.x ACLs    | UNIX                |                  |
        | Volumes)  |             |-----------------|---------------------|                  |
        |           |             | NTFS ACLs       | NTFS                |                  |
        +------------------------------------------------------------------------------------+
```

Where and when to set security style
How to decide on what security style to use on Vservers with FlexVol volumes
```
	+-------------------------------------------------------------------------------------------------+
	| Security style | Choose if...                                                                   |
	|----------------|--------------------------------------------------------------------------------|
	| UNIX           | The file system is managed by a UNIX administrator.                            |
	|                | The majority of users are NFS clients.                                         |
	|                | An application accessing the data uses a UNIX user as the service account.     |
	|----------------|--------------------------------------------------------------------------------|
	| NTFS           | The file system is managed by a Windows administrator.                         |
	|                | The majority of users are CIFS clients.                                        |
	|                | An application accessing the data uses a Windows user as the service account.  |
	|----------------|--------------------------------------------------------------------------------|
	| Mixed          | The file system is managed by both UNIX and Windows administrators and         |
	|                | +users consist of both NFS and CIFS clients.                                   |
	+-------------------------------------------------------------------------------------------------+
```

```
	How security style inheritance works
	If you do not specify the security style when creating a new FlexVol volume or qtree, it inherits its security style.
	Security styles are inherited in the following manner:
	* A FlexVol volume inherits the security style of the root volume of its containing Vserver.
	* A qtree inherits the security style of its containing FlexVol volume.
	* A file or directory inherits the security style of its containing FlexVol volume or qtree.
	Infinite Volumes cannot inherit security styles. All files and directories in Infinite Volumes always
	use the unified security style. The security style of an Infinite Volume and the files and directories it
	contains cannot be changed
```

Comparison of FlexVol volumes and Infinite Volumes
#https://library.netapp.com/ecmdocs/ECMP1196906/html/GUID-58EC88D8-29F9-4865-88B9-A4FB66224FEE.html
```
	Both FlexVol volumes and Infinite Volumes are data containers. However, they
	have significant differences that you should consider before deciding which
	type of volume to include in your storage architecture.

	The following table summarizes the differences and similarities between FlexVol
	volumes and Infinite Volumes:

	+------------------------------------------------------------------------------+
	|     Volume      |  FlexVol volumes   | Infinite Volumes |       Notes        |
	|  capability or  |                    |                  |                    |
	|     feature     |                    |                  |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Containing      | Vserver; single    | Vserver; can     |                    |
	| entity          | node               | span nodes       |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Number of       | One                | Multiple         |                    |
	| associated      |                    |                  |                    |
	| aggregates      |                    |                  |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Maximum size    | 32-bit volumes: 16 | Up to 20 PB      | For information    |
	|                 | TB                 |                  | about the maximum  |
	|                 |                    |                  | size of 64-bit     |
	|                 | 64-bit volumes:    |                  | volumes, see the   |
	|                 | model-dependent    |                  | Hardware Universe. |
	|-----------------+--------------------+------------------+--------------------|
	| Minimum size    | 20 MB              | Approximately    |                    |
	|                 |                    | 1.33 TB for each |                    |
	|                 |                    | node used        |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Type of Vserver | Vserver with       | Vserver with     |                    |
	|                 | FlexVol volume     | Infinite Volume  |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Maximum number  | Model- and         | One              | For more           |
	| per Vserver     | protocol-dependent |                  | information, see   |
	|                 |                    |                  | the Hardware       |
	|                 |                    |                  | Universe.          |
	|-----------------+--------------------+------------------+--------------------|
	| Maximum number  | Model-dependent    | Model-dependent  | For more           |
	| per node        |                    |                  | information, see   |
	|                 |                    |                  | the Hardware       |
	|                 |                    |                  | Universe.          |
	|-----------------+--------------------+------------------+--------------------|
	| Types of        | 64-bit or 32-bit   | 64-bit           |                    |
	| aggregates      |                    |                  |                    |
	| supported       |                    |                  |                    |
	|-----------------+--------------------+------------------+--------------------|
	| SAN protocols   | Yes                | No               |                    |
	| supported       |                    |                  |                    |
	|-----------------+--------------------+------------------+--------------------|
	| File access     | NFS, CIFS          | NFS, CIFS        | For more           |
	| protocols       |                    |                  | information, see   |
	| supported       |                    |                  | the Clustered Data |
	|                 |                    |                  | ONTAP File Access  |
	|                 |                    |                  | and Protocols      |
	|                 |                    |                  | Management Guide.  |
	|-----------------+--------------------+------------------+--------------------|
	| Deduplication   | Yes                | Yes              |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Compression     | Yes                | Yes              |                    |
	|-----------------+--------------------+------------------+--------------------|
	| FlexClone       | Yes                | No               |                    |
	| volumes         |                    |                  |                    |
	|-----------------+--------------------+------------------+--------------------|
	| FlexCache       | Yes                | No               |                    |
	| volumes         |                    |                  |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Quotas          | Yes                | No               |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Qtrees          | Yes                | No               |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Thin            | Yes                | Yes              |                    |
	| provisioning    |                    |                  |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Snapshot copies | Yes                | Yes              |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Data protection | Yes                | Yes              | For Infinite       |
	| mirrors         |                    |                  | Volumes, only      |
	|                 |                    |                  | mirrors between    |
	|                 |                    |                  | clusters are       |
	|                 |                    |                  | supported.         |
	|-----------------+--------------------+------------------+--------------------|
	| Load-sharing    | Yes                | No               |                    |
	| mirrors         |                    |                  |                    |
	|-----------------+--------------------+------------------+--------------------|
	| Tape backup     | Yes                | Yes              | For Infinite       |
	|                 |                    |                  | Volumes, you must  |
	|                 |                    |                  | use NFS or CIFS    |
	|                 |                    |                  | rather than NDMP.  |
	|-----------------+--------------------+------------------+--------------------|
	| Volume security | UNIX, NTFS, mixed  | Unified          | For more           |
	| styles          |                    |                  | information, see   |
	|                 |                    |                  | the Clustered Data |
	|                 |                    |                  | ONTAP File Access  |
	|                 |                    |                  | and Protocols      |
	|                 |                    |                  | Management Guide.  |
	+------------------------------------------------------------------------------+
```

TIPs 删除volume: 先offline再删除
```
volume offline -vserver nfs-ds -volume nfsshare
volume delete -vserver nfs-ds -volume nfsshare
```
