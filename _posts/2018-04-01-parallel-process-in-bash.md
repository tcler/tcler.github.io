---
layout: post
title: "bash 并行(parallel-process-in-bash)"
---
    
### 方案1
Use wait built-in command with &
```
while read url; wget -q $url &; done <url.list
wait

# 缺点: 任务的stdout/stderr会混在一起; 没有并行任务数量限制
#       都需要自己编码解决
# 相似方案: 使用 screen ，不过使用 screen 就不能用 wait 了
```

### 方案2
Use xargs command
```
xargs -n 1 -P 4 -I{} wget -q {} < list.txt
# Use the -n option or the -L option with -P; otherwise chances are that only one exec will be done.

# 缺点: 任务的stdout/stderr会混在一起
# 优点: 可以用 -P 选项限制并行任务数量
```

### 方案3
Use GNU/parallel command
```
parallel -j 4 wget -q {} < list.txt

# 缺点: 有些 OS 好像默认没有这个包，RHEL/CentOS 需要加 epel 源:
# yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm
```

### 方案4
Use tmux (or screen, note: screen is deprecated)
```
tmux new -s session_name1 -d "command line 1"
tmux new -s session_name2 -d "command line 2"
tmux new -s session_name3 -d "command line 3"
tmux new -s session_name4 -d "command line 4"
```

```
screen -S session_name -dm bash -c "command"
```
