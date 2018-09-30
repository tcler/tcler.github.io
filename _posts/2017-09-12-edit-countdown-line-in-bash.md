---
layout: post
title: "edit countdown line how to in bash"
---

### edit from first line to countdown 5th line
sed: sorry I don't support countdown/negative line number

ed/ex: it doesn't matter, we can!

#bash 脚本里，偶尔会遇到处理文件倒数第几行的需求，然而 sed 不支持倒数第几行的操作，，

#没有关系我们还有古老的 ed/ex 可以完美解决

```
ex -s <(seq 1 10)  <<< $'1,-5 s/$/,/ \n 1,$p \n'
```
