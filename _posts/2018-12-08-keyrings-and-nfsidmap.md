---
layout: post
title: "keyrings and nfsidmap"
---

```
* key retention 服务简介
  Linux 密钥保留服务(Linux key retention service) 是在 Linux-2.6 中引入的一个特性

  该服务用于在 Linux 内核中缓存 加密密钥，身份验证令牌，跨域用户映射 等信息提供给
  文件系统 和 其他内核服务 使用。

  它使得 Linux 内核能够快速访问所需的密钥等信息，并提供 API 可以从用户态操作密钥
  (比如添加、更新和删除)


* 到底啥是 key
  cryptographic data, authentication tokens, keyrings, etc.. 都属于 key ,
  在代码里都通过 struct key 来表示

  每个 key 包括如下属性:

	- A serial number.
	- A type.
	- A description (for matching a key in a search).
	- Access control information.
	- An expiry time.
	- A payload.
	- State.

  # keyring 是一种特殊类型的key，用来保存到(指向)其他密钥的链接。每个进程都有三个
  # 标准的 keyring 订阅，内核服务可以通过它们查找相关的key。


* Key types (see keyrings(7))
  The kernel provides several basic types of key:

       "keyring"
              Keyrings are special keys which store a set of links to other
              keys (including other keyrings), analogous to a directory
              holding links to files.  The main purpose of a keyring is to
              prevent other keys from being garbage collected because
              nothing refers to them.

              Keyrings with descriptions (names) that begin with a period
              ('.') are reserved to the implementation.

       "user" This is a general-purpose key type.  The key is kept entirely
              within kernel memory.  The payload may be read and updated by
              user-space applications.

              The payload for keys of this type is a blob of arbitrary data
              of up to 32,767 bytes.

              The description may be any valid string, though it is
              preferred that it start with a colon-delimited prefix
              representing the service to which the key is of interest (for
              instance "afs:mykey").

       "logon" (since Linux 3.3)
              This key type is essentially the same as "user", but it does
              not provide reading (i.e., the keyctl(2) KEYCTL_READ
              operation), meaning that the key payload is never visible from
              user space.  This is suitable for storing username-password
              pairs that should not be readable from user space.

              The description of a "logon" key muststart with a non-empty
              colon-delimited prefix whose purpose is to identify the
              service to which the key belongs.  (Note that this differs
              from keys of the "user" type, where the inclusion of a prefix
              is recommended but is not enforced.)

       "big_key" (since Linux 3.13)
              This key type is similar to the "user" key type, but it may
              hold a payload of up to 1 MiB in size.  This key type is
              useful for purposes such as holding Kerberos ticket caches.

              The payload data may be stored in a tmpfs filesystem, rather
              than in kernel memory, if the data size exceeds the overhead
              of storing the data in the filesystem.  (Storing the data in a
              filesystem requires filesystem structures to be allocated in
              the kernel.  The size of these structures determines the size
              threshold above which the tmpfs storage method is used.)
              Since Linux 4.8, the payload data is encrypted when stored in
              tmpfs, thereby preventing it from being written unencrypted
              into swap space.


* 自定义 key 类型
  内核服务可能希望定义自己的密钥类型。 例如，AFS filesystem 可能希望定义新的
  key type 来保存 Kerberos 5 ticket 信息;
  创建自定义 key type 很简单，只需要 初始化 key_type struct 结构体然后调用 API
  int register_key_type(struct key_type *type); 进行注册就可以了


* nfs4 idmap 自定义的 key type: "id_resolver"
  RHEL-6.3 之后的 nfs-utils 增加了基于 keyrings 机制的 idmap 查找方式，主要是为了
  改善 idmap 转换效率?(不需要每次查找都向用户态 upcall)
  
  # The NFSv4 client instead uses nfsidmap(8), and only falls back to rpc.idmapd
    if there was a problem running the nfsidmap(8) program.
  ? 为啥 NFSv4 server 不用 nfsidmap(8) ?? //问问开发去

  '''
  $ sed -rn '/struct key_type key_type_id_resolver = /,/^};/ p' nfs/nfs4idmap.c
  static struct key_type key_type_id_resolver = {
  	.name		= "id_resolver",
  	.instantiate	= user_instantiate,
  	.match		= user_match,
  	.revoke		= user_revoke,
  	.destroy	= user_destroy,
  	.describe	= user_describe,
  	.read		= user_read,
  };

  # call stack:
    module_init(init_nfs_v4)
      init_nfs_v4 ->
        nfs_idmap_init ->
          register_key_type(&key_type_id_resolver)
   '''

* 查看 id_resolver 类型的 key
  '''
  ~]$ sudo nfsidmap -d
  domain.com
  ~]$ sudo nfsidmap -l
  No .id_resolver keys found.
  '''

* nfs4 idmap is not used for AUTH_SYS
  see:
    https://serverfault.com/questions/535809/nfsv4-with-idmap
    https://www.spinics.net/lists/linux-nfs/msg38598.html
  
  所以想要看效果必须配置 krb5 ,,
```
