---
layout: post
title: "[转] Study Linux Team Driver"
tags: [linux, team]
author: github.com/ceyes
---

## What is Team Driver

Team driver 是Linux 3.3 加入的新feature，[Kernel Newbies](http://kernelnewbies.org/Linux_3.3) 对其是这样介绍的

>1.4. Better bonding of network interfaces: teaming
>
>There is a new "teaming" network device, which is intended to be a fast, scalable, clean, userspace-driven replacement for the bonding driver. It allows to create virtual interfaces that teams together multiple Ethernet devices. This is typically used to increase the maximum bandwidth and provide redundancy. Currently round-robin and active-backup modes are implemented. The libteam userspace library with couple of demo apps is available at <https://github.com/jpirko/libteam>
>
>Code: [commit](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=3d249d4ca7d0ed6629a135ea1ea21c72286c0d80)

作者在其项目主页[libteam.org](http://libteam.org/)的介绍是：

>The purpose of the Team driver is to provide a mechanism to team multiple NICs (ports) into one logical one (teamdev) at L2 layer. The process is called "channel bonding", "Ethernet bonding", "channel teaming", "link aggregation", etc. This is already implemented in the Linux kernel by the bonding driver.
>
>One thing to note is that Team driver project does try to provide the similar functionality as the bonding driver, however architecturally it is quite different from bonding driver. Team driver is modular, userspace driven, very lean and efficient, and it does have some distinct advantages over bonding. The way Team is configured differs dramatically from the way bonding is.
>
>Note that there is no performance overhead added with the fact that the control logic happens in userspace. Packets never leave kernel. 

所以它是 bonding 的另一种实现，作用依然是链路聚合——把几个真实的网卡绑定成一个虚拟的二层接口。功能不外乎是增加带宽（MT）和高可用性（HA）。虽然功能和bonding相似，但实现和配置都有很大的不同，作者强调Team 是“userspace driven”，非常精简和高效，而且性能开销并没有因此增加。

### Feature

作者列出的表[Bonding vs. Team driver features](https://github.com/jpirko/libteam/wiki/Bonding-vs.-Team-driver-features)中详细对比了Team和Bonding的features。  
让我印象深刻的主要有：

* user-space runtime control - 配置主要在user space进行，灵活性、扩展性随之提高
* NS/NA (IPV6) link monitoring - 比Bonding的两种链路检测（miimon 和 ARP monitor）多了IPv6 的NS/NA 方式
* separate per-port link monitoring setup - 每个端口都能配置不同的链路检测
* Highly customizable hash function setup - 可定制的负载均衡策略

### Component

因为Team尽量把不需要kernel处理的工作拿到了user space。所以组件会多一些，主要有3部分:

1.  Team driver (kernel)

    非常重要的部分——内核模块。主要工作就是收发包（skb flows）,本身没有控制逻辑，具体的策略行为都由user space 的程序来设定。

    ```
     # modinfo team
    filename:       /lib/modules/3.10.0-123.el7.x86_64/kernel/drivers/net/team/team.ko
    alias:          rtnl-link-team
    description:    Ethernet team device driver
    author:         Jiri Pirko <jpirko@redhat.com>
    license:        GPL v2
    srcversion:     39F7B52A85A880B5099D411
    depends:        
    intree:         Y
    vermagic:       3.10.0-123.el7.x86_64 SMP mod_unload modversions 
    signer:         Red Hat Enterprise Linux kernel signing key
    sig_key:        00:AA:5F:56:C5:87:BD:82:F2:F9:9D:64:BA:83:DD:1E:9E:0D:33:4A
    sig_hashalgo:   sha256

    ```

2.  libteam (userspace)

    Lib, 封装了和 Team driver 通信的API（Netlink……），用以帮助写Team应用程序

    ```
    # rpm -ql libteam | grep -E "/usr/[bin|lib]"
    /usr/bin/teamnl
    /usr/lib64/libteam.so.5
    /usr/lib64/libteam.so.5.0.0

    ```

3.  teamd (userspace)

    teamd 代表 "Team daemon"，用以创建 Team 实例，并且设定Team的行为，就像bonding里的"mode"，作者把它叫做"runners"。  
    teamd 不是必需的，用户可以借助libteam写自己的程序。

    ```
     # rpm -ql teamd | grep -E "/usr/[bin|lib]"
    /usr/bin/bond2team
    /usr/bin/teamd
    /usr/bin/teamdctl
    /usr/lib/systemd/system/teamd@.service
    /usr/lib64/libteamdctl.so.0
    /usr/lib64/libteamdctl.so.0.1.0
    ```

## Setup / Configuration

配置Team的方法有很多，可以使用teamd 或 直接调 driver API。

### Setup by teamd + teamctl (推荐用法)

*   Create team instance

    `teamd -d` 就会产生 team 接口，`-d` 表示 daemonize。
    默认接口名是 team0，可以使用 `-t <DEVNAME>` 指定成特定的名字。
    更多用法参考 `teamd -h`。

    对于 team 以及 ports 的 参数设定可以用 `-f <config-file>` 或 `-c <config-string>` 来指定。
    作者提供了不少 config-file 的 sample 在 `/usr/share/doc/teamd-*/example_configs/`; 而所有的选项以及 JSON-config-string 写法可以通过`man teamd.conf` 来学习; 此外作者还提供了一个把 bonding 的配置转换成 team 配置文件的工具 `bond2team`。

    对 ports 的设置可以在后面用 teamdctl 来更新或修改，而对 team 的设置应该在这一步指定好，比如重要的 runner 和 link_watch。

    Sample:

    ```
    # teamd -t team0 -d -c '{ "runner" : { "name": "activebackup" },  "link_watch" : { "name": "ethtool" } }'
    ```

    这个 JSON 格式有点难以接受，很不方便在Bash下编辑和处理，比起 bonding parameter 的 `key=val` 写法要麻烦太多。

*   Play with teamdctl

    `teamdctl` 提供查看team、设置team(很少一部分)、添加port、设置port 等功能。
    Refer `teamdctl help`

    ```
    # teamdctl team0 state view
    # teamdctl team0 state dump
    # teamdctl team0 config dump
    # teamdctl team0 port config dump em1
    # teamdctl team0 port add em1
    # teamdctl team0 port remove em1
    # teamdctl team0 port config update em1 JSON-config-string
    ```
    要注意的是：添加 port 前需要先把该网卡 link down.


*   Kill team instance

    ```
    # teamd -t team0 -k
    ```

### Setup by `ip utility` + `teamnl` (directly call Team Netlink API)

*   Create or delete team interface:

    ```
    # ip link add name team0 type team
    # ip link delete team0
    ```

*   To add/remove a port em1 to a network team team0

    ```
    # ip link set dev em1 down
    # ip link set dev em1 master team0
    # ip link set dev em1 nomaster
    ```
    Team driver will bring ports up automatically.

*   Listing the ports of team0

    ```
    # teamnl team0 ports
    ```

*   Querying / Configuring team options

    ```
    # teamnl team0 options
    # teamnl team0 setoption mode activebackup
    # teamnl team0 getoption activeport
    # teamnl team0 setoption activeport 5
    # teamnl team0 getoption activeport
    ```

## ref:

* <https://github.com/jpirko/libteam/wiki/Infrastructure-Specification>
* <http://libteam.org/>
* [RHEL7 Team Documentation](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/ch-Configure_Network_Teaming.html)
