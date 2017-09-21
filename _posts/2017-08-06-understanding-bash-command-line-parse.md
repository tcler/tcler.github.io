---
layout: post
title: "理解 bash 命令行解析过程 - 单词分割"
---

### bash 基本功能
从命令行读取一个字符串，按照规则将字符串分割成一个字符串列表(数组)，然后将第一个字符串当成 命令(或函数)，其余的当作这个命令的参数; 然后尝试执行该命令。

### IFS (Internal  Field Separator)
既然是单词分割，就必须要有分割符(Internal  Field Separator)
```
IFS    The  Internal  Field Separator that is used for word splitting after expansion and to split lines into words with the read builtin command.
       The default value is ``<space><tab><newline>''.
```

### 如果我的参数里本身就有 IFS 里的字符比如'空格'怎么办?
```
[yjh@ws tmp]$ ls
我是一个 带有空格 的文件.txt
[yjh@ws tmp]$ ls -l 我是一个 带有空格 的文件.txt
ls: 无法访问'我是一个': No such file or directory
ls: 无法访问'带有空格': No such file or directory
ls: 无法访问'的文件.txt': No such file or directory
```

bash 提供了三种方法 \  ""  '' 来解决这个问题:
```
[yjh@ws tmp]$ ls -l 我是一个\ 带有空格\ 的文件.txt
-rw-rw-r--. 1 yjh yjh 0 9月  21 16:02 我是一个 带有空格 的文件.txt
[yjh@ws tmp]$ ls -l "我是一个 带有空格 的文件.txt"
-rw-rw-r--. 1 yjh yjh 0 9月  21 16:02 '我是一个 带有空格 的文件.txt'
[yjh@ws tmp]$ ls -l '我是一个 带有空格 的文件.txt'
-rw-rw-r--. 1 yjh yjh 0 9月  21 16:02 '我是一个 带有空格 的文件.txt'
```

为啥是三种，选择恐惧症怎么办? 其实三种方式在特定场景不能相互替代:

\\
```
command with --many --command --line --options and many long file path \
   /usr/local/share/document/2017-08-06-understanding-bash-command-line-parse.md \
   /usr/local/share/document/2017-08-14-understanding-bash-redirect.md
```
""
```
command with a "parameter include ' ' and single quota '' "
command with a "parameter include $variable_substitution and $(command substitution)"
```
''
```
command with a 'parameter include char $string and $(string)'
```
