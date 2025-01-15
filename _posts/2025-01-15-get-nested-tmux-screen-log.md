---
layout: post
title: "get nested tmux screen log"
---

# What happen: pipe-pane sub-command get wrong log while tmux run in nested
Before this, I used the following method to grab the screen log of the tmux dettach session and it has been working well:
```
session=console-$vmname
tmux new -s $session -d vm console $vmname \; pipe-pane "cat >$logpath/console-$vmname.log"
```
But when I tried to run the test script in tmux instead in normal terminal, I found that the virsh console log is not expected.
The captured log from **pipe-pane** becomes the standard output of the script itself instead the real VM console log.

# workaround or resolution
After many tests and confirmations, we first eliminated the bug in bash itself; the problem should be caused by the default 
behavior of tmux pipe-pane itself. Then I checked the tmux man page and found that the pipe-pane command has a **-t** option, which 
can be used to specify a target pane; then after searching and tring I found that the pane name/ID corresponding to a single session
is **${session}:0.0**. So the final solution is:
```
session=console-$vmname
tmux new -s $session -d vm console $vmname \; pipe-pane -t ${session}:0.0 "cat >$logpath/console-$vmname.log"
```
