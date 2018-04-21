---
layout: post
title: "从 git repo 获取子目录内容"
---

### git archive
```
git archive --remote=${gitrepo} <branch> a/b/c | tar xvf -
```

不过 git archive 不支持 http/https 协议，所以如果 github 下载 subdir 就不行了

### svn export
```
# must replace 'tree/master' or 'blob/master' with trunk
# svn export ${gitpath/????\/master/trunk} [new_path_name]
$ gitpath=https://github.com/hpc/lustre/tree/master/lustre/tests/racer
$ LANG=C svn export ${gitpath/????\/master/trunk} lustre-racer.orig
A    lustre-racer.orig
A    lustre-racer.orig/dir_create.sh
A    lustre-racer.orig/file_concat.sh
A    lustre-racer.orig/file_create.sh
A    lustre-racer.orig/file_exec.sh
A    lustre-racer.orig/file_link.sh
A    lustre-racer.orig/file_list.sh
A    lustre-racer.orig/file_rename.sh
A    lustre-racer.orig/file_rm.sh
A    lustre-racer.orig/file_symlink.sh
A    lustre-racer.orig/racer.sh
Exported revision 11168.
```
