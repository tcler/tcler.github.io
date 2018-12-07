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


* 三种特殊 key 类型
  The key service defines three special key types:

     (+) "keyring"

	 Keyrings are special keys that contain a list of other keys. Keyring
	 lists can be modified using various system calls. Keyrings should not
	 be given a payload when created.
	 # keyring 其实就是一个保存 key 的容器

     (+) "user"

	 A key of this type has a description and a payload that are arbitrary
	 blobs of data. These can be created, updated and read by userspace,
	 and aren't intended for use by kernel services.
	 # 一个包含 description 和 payload 的数据集合，由用户态维护使用

     (+) "logon"

	 Like a "user" key, a "logon" key has a payload that is an arbitrary
	 blob of data. It is intended as a place to store secrets which are
	 accessible to the kernel but not to userspace programs.
	 # 和 user 类型类似，但是 payload 只能由内核空间访问

	 The description can be arbitrary, but must be prefixed with a non-zero
	 length string that describes the key "subclass". The subclass is
	 separated from the rest of the description by a ':'. "logon" keys can
	 be created and updated from userspace, but the payload is only
	 readable from kernel space.
	 # description 必须有一个 $subclass: 前缀，可以从userspace创建和更新
	 # 但是 payload 只能从内核空间读取

* 自定义 key 类型
  内核服务可能希望定义自己的密钥类型。 例如，AFS filesystem 可能希望定义新的
  key type 来保存 Kerberos 5 ticket 信息;
  创建自定义 key type 很简单，只需要 初始化 key_type struct 结构体然后调用 API
  int register_key_type(struct key_type *type); 进行注册就可以了

* nfs4 idmap 自定义的 key type: "id_resolver"
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
