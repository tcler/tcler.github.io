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

#get super block info: blocksize
SB=$(xfs_db -r $dev -c "inode 0" -c "type sb" -c print | sed -nr '/dblocks|blocksize|inodesize/{s/ //g;p}')
eval "$SB"

#get inode info: file size and extents list
INODE=$(xfs_db -r $dev -c "inode $inum" -c "type inode" -c print)
fsize=$(awk '/core.size/{print $NF}' <<<"$INODE")
extents=$(awk -F'[0-9]:' '/u.bmx/{for(i=2;i<=NF;i++){print $i}}' <<<"$INODE")

#output file content to stdout
left=$fsize
while read extent; do
        echo "extent: $extent" >&2
        read startoff startblock blockcount extentflag  <<< ${extent//[,\][]/ }
        ddcount=$blockcount

        if [[ $((ddcount * blocksize)) -gt $left ]]; then
                ddcount=$((left/blocksize))
                mod=$((left%blocksize))

                echo "left=$left, extentSize=$((ddcount * blocksize)); ddcount=$ddcount, mod=$mod" >&2
                dd if=$dev bs=$blocksize skip=$startblock count=$ddcount
                dd if=$dev bs=1 skip=$(((startblock+ddcount)*blocksize)) count=$mod

                left=0
                break
        else
                dd if=$dev bs=$blocksize skip=$startblock count=$ddcount
        fi
        ((left-=(ddcount*blocksize)))
done <<< "$extents"

echo "left: $left" >&2
```
