---
layout: post
title: "Open/read file by inode"
---

### 问题/需求
看到一个匿名文件但有 inode number (或者文件被删除了但是我们记下了它的 inode number）；

怎么去读该文件的内容呢(假设文件占用的块还没有被重新分配出去)？


### 解决方案 (just for ext4/3/2 and xfs now)
ext* 文件系统用户态工具 debugfs 里有一个 cat 命令 可以直接指定 inode 把文件内容打印出来

There's a sub-command cat in debugfs(tool for ext* filesystem) can do this
```
[yjh@test ~]$ ls -l /proc/15881/fd
总用量 0
lrwx------. 1 yjh yjh 64 1月   5 16:37 0 -> /dev/pts/2
lrwx------. 1 yjh yjh 64 1月   5 16:37 1 -> /dev/pts/2
lrwx------. 1 yjh yjh 64 1月   5 16:37 2 -> /dev/pts/2
lrwx------. 1 yjh yjh 64 1月   5 16:37 3 -> /home/yjh/tmp/#786479 (deleted)
[yjh@test ~]$ sudo debugfs -R 'cat <786479>' /dev/mapper/fedora-home 2>/dev/null
hello would, I love you ...
```

但是 xfs 文件系统，xfs_* 命令里没有找到相应的功能，自己写一个:

But seems there's no function like debugfs->cat in xfs_* tools,
 so I wrote a script for it based on xfs_db:
```
#!/bin/bash
#auth: yin-jianhong@163.com

dev=$1
inum=$2

[[ -z "$inum" ]] && {
        echo "Usage: sudo xfs_icat <dev> <inum>" >&2
}

#get super block info: blocksize, $inum core.size and extents list
INFO=$(xfs_db -r $dev -c "inode 0" -c "type sb" -c 'print blocksize' -c "inode $inum" -c "type inode" -c "print core.size u.bmx")
{
read k eq blocksize
read k eq fsize
read k eq sum extents
} <<< "$INFO"

#output file content to stdout
left=$fsize
for extent in $extents; do
        echo "extent: $extent" >&2
        read idx startoff startblock blockcount extentflag  <<< ${extent//[:,\][]/ }
        extentSize=$((ddcount * blocksize))
        ddcount=$blockcount

        if [[ $extentSize -gt $left ]]; then
                ddcount=$((left/blocksize))
                mod=$((left%blocksize))

                echo "left=$left, extentSize=$extentSize; ddcount=$ddcount, mod=$mod" >&2
                dd if=$dev bs=$blocksize skip=$startblock count=$ddcount
                dd if=$dev bs=1 skip=$(((startblock+ddcount)*blocksize)) count=$mod
                break
        else
                dd if=$dev bs=$blocksize skip=$startblock count=$ddcount
                echo >&2
        fi

        ((left-=(ddcount*blocksize)))
done
```

```
# 经同事提醒，目前这个脚本只能处理 core.format = 2 (extents) 的情况
另外两种情况
    core.format = 3 (btree)
    core.format = 1 (local)
还有 core.version 的值不同，相应的 field 名字也不一样，需要做判断处理
-可能需要借助 expect ? 以交互式方式执行
具体 xfs_db example 可以参考: https://github.com/djwong/xfs-documentation

(tip: 制作 core.format 为 "3 (btree)" 的方法，可以创建一个 xfs.image，
  然后在里面写满4k大小的文件，然后每隔一个删除，然后再创建一个大文件 
  就可以得到一个 btree 格式的 inode 了)
```
