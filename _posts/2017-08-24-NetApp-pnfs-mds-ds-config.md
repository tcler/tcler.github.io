---
layout: post
title: "NetApp pnfs mds ds 配置"
---

```
The way to know which is the MDS or the DS is depending on the configuration for each vserver.
To use pNFS on a particular vserser, the vserver should have at least two data LIFs on different nodes.
This is my configuration for one of my vsersers:

mora1::> network interface show -vserver vs1
            Logical    Status     Network            Current       Current Is
Vserver     Interface  Admin/Oper Address/Mask       Node          Port    Home
----------- ---------- ---------- ------------------ ------------- ------- ----
vs1
            data1        up/up    192.168.0.61/24    mora1-01      e0d     true
            data2        up/up    192.168.0.71/24    mora1-02      e0d     true
2 entries were displayed.


Notice vserver vs1 has two LIFs. Now look at the volumes I have for vserver vs1:

mora1::> volume show -vserver vs1
Vserver   Volume       Aggregate    State      Type       Size  Available Used%
--------- ------------ ------------ ---------- ---- ---------- ---------- -----
vs1       rootvol1     aggr_1       online     RW          1GB    971.9MB    5%
vs1       vol1         aggr_01      online     RW          6GB     5.70GB    5%
vs1       vol2         aggr_02      online     RW         10GB     9.50GB    5%
3 entries were displayed.


Of course volume rootvol1 is the root of the vserver so I created two volumes: vol1 and vol2,
where vol1 is in aggregate aggr_01 and vol2 is in aggregate aggr_02. Now look at the aggregates:

mora1::> storage aggregate show -aggregate aggr_01,aggr_02                                                                  
Aggregate     Size Available Used% State   #Vols  Nodes            RAID Status
--------- -------- --------- ----- ------- ------ ---------------- ------------
aggr_01     7.03GB    1021MB   86% online       1 mora1-01         raid_dp,
                                                                   normal
aggr_02    10.55GB   502.8MB   95% online       1 mora1-02         raid_dp,
                                                                   normal
2 entries were displayed.


With this information, vol1 (junction path /vol1) which is in aggregate aggr_01 which in turn this
aggregate in owned by node mora1-01 and this node has a data LIF data1 (192.168.0.61),
therefore mounting vol1 (using each of the data LIFs):

192.168.0.61:/vol1 – MDS:192.168.0.61 (server in mount command), DS:192.168.0.61 (vol1 is homed in 192.168.0.61)
192.168.0.71:/vol1 – MDS:192.168.0.71 (server in mount command), DS:192.168.0.61 (vol1 is homed in 192.168.0.61)

For volume vol2 (junction path /vol2) which is in aggregate aggr_02 which in turn this
aggregate is owned by node mora1-02 and this node has a data LIF data2 (192.168.0.71),
therefore mounting vol2 (using each of the data LIFs):

192.168.0.61:/vol2 – MDS:192.168.0.61 (server in mount command), DS:192.168.0.71 (vol2 is homed in 192.168.0.71)
192.168.0.71:/vol2 – MDS:192.168.0.71 (server in mount command), DS:192.168.0.71 (vol2 is homed in 192.168.0.71)

In conclusion, to use pNFS you need at least two LIFs owned by different nodes and the MDS is always
the LIF you specify in the mount command and the DS is always the LIF where the volume you are mounting
is physically located.

Hope this helps.
```

以上是NetApp的工程师的描述，简单总结一下
```
NetApp MDS&DS configration point: "vserver are both MDS and DS"
如果需要测试MDS和DS，需要在一个VSM(vserser)上至少配置两对儿 LIF+volum，并且两个LIF分别属于两个Node
+------------------------------+
| vserver1                     |
|                              |
| vol1                    vol2 |
| LIF1(IP1)          (IP2)LIF2 |
+--|-----------------------|---+
   |                       |
+--v-----+            +----v---+
| Node1  |            | Node2  |
+--------+            +--------+

mount IP1:/vol2 $nfsmp; dd if=/dev/zero of=$nfsmp/testfile bs=1b count=10000
 抓包查看数据走的是不是 DS IP2
mount IP2:/vol1 $nfsmp; dd if=/dev/zero of=$nfsmp/testfile bs=1b count=10000
 抓包查看数据走的是不是 DS IP2
```

ref:
```
http://www.netapp.com/us/media/tr-4063.pdf
https://library.netapp.com/ecm/ecm_download_file/ECMP1196891
  [downloaded] https://wiki.test.redhat.com/Kernel/Filesystem/windows-netapp?action=AttachFile&do=view&target=Clustered_Data_ONTAP_82_File_Access_and_Protocols.pdf
https://library.netapp.com/ecm/ecm_download_file/ECMP1610208
https://library.netapp.com/ecmdocs/ECMP1196798/html/GUID-D860985F-11BF-4589-9457-E0FD03130B1F.html
```
