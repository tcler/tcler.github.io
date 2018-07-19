---
layout: post
title: "Fedora-28 glibc/regex 一个 早有预谋的 更改"
---

### 问题

Zorro 这两天发现一个测试套件在新的系统上很诡异的安装失败了，调查发现是 glibc 更新了 regex 库导致的;
具体表现就是 grep [a-z] 匹配不能得到预期结果，大写字母也会匹配到；但上游开发确说这是预期结果，又因为信息不对称开发给出的基于权重算法的解释
我们并没有看懂~
```
[yjh@test nfs]$ echo ABCD | grep -o [a-z]
A
B
C
D
```

### 调查分析
先通过猜测: 是否跟 locale 设置有关? 然后添加 LANG=C 验证，果然得到预期结果
```
[yjh@test nfs]$ echo ABCDa | LANG=C grep -o [a-z]
a
```

然后翻看 man page 才发现不止最新的 Fedora-28, 甚至 RHEL-6 系统的 man grep 都有相同的描述
```
[root@bkr-host ~]# lsb_release -sir
RedHatEnterpriseWorkstation 6.10
[root@bkr-host ~]# man grep | col -bx | grep 'Character Classes and Bracket Expressions' -A 17
   Character Classes and Bracket Expressions
       A bracket expression is a list of characters enclosed by [ and  ].   It
       matches  any  single  character in that list; if the first character of
       the list is the caret ^ then it matches any character not in the  list.
       For  example,  the  regular  expression [0123456789] matches any single
       digit.

       Within a  bracket  expression,  a  range  expression  consists  of  two
       characters separated by a hyphen.  It matches any single character that
       sorts  between  the  two  characters,  inclusive,  using  the  locale’s
       collating  sequence  and  character set.  For example, in the default C
       locale, [a-d] is equivalent to [abcd].  Many locales sort characters in
       dictionary   order,  and  in  these  locales  [a-d]  is  typically  not
       equivalent to [abcd]; it might be equivalent to [aBbCcDd], for example.
       To  obtain  the  traditional interpretation of bracket expressions, you
       can use the C locale by setting the LC_ALL environment variable to  the
       value C.
```

原来这次的 glibc 修改是 "早有预谋" 的 ...
原来我们一直以来对 [a-z] [A-Z] 的用法是 "靠巧合编程", 能得到预期结果的前提是必须 LC_ALL=C; 
其他 locale 设置，[a-d] 的预期结果可能是 [abcd] 也可能 [aBbCcDd] 还可能 [aAbBcCd] ...
```
[yjh@test nfs]$ echo {a..z} {A..Z} | grep [a-d] -o | xargs
a b c d A B C
```

### 更近一步调查验证
根据 man page 描述 "Many locales sort characters in dictionary order, ", 所以我们可以
猜测关于这次修改: 至少我们用到的 en_US.utf8 zh_CN.utf8 的字符字典顺序被修改了:

原先是 A .. Z -> a .. z , 而现在改成 aA bB  ..  zZ

再结合 "Fedora 28 Update: glibc-2.27-30.fc28"[1] 中的 bug 1582229[2] 的描述，我们猜测排序里不止包含 aA
而是更多 比如 "aAÀâ" "bBB`" ... , 验证如下:
- [1] https://www.spinics.net/lists/fedora-package-announce/msg247214.html
- [2] https://bugzilla.redhat.com/show_bug.cgi?id=1582229
```
[yjh@test nfs]$ echo "AaÀâ"|grep -o [[=a=]]
A
a
À
â
[yjh@test nfs]$ echo "AaÀâ"|LANG=C grep -o [[=a=]]
a
[yjh@test nfs]$ echo "AaÀâ"| grep -o '[A-a]'   # a is before A
grep: Invalid range end
[yjh@test nfs]$ echo "AaÀâ"| grep -o '[a-A]'
A
a
[yjh@test nfs]$ echo "AaÀâ"| grep -o '[a-À]'   # À is before â
A
a
À
[yjh@test nfs]$ echo "AaÀâ"| grep -o '[a-â]'
A
a
À
â
```

### 总结
以后些正则最好使用 [:alnum:], [:alpha:],[:cntrl:], [:digit:], [:graph:], [:lower:],
[:print:], [:punct:], [:space:], [:upper:], and [:xdigit:] 来代替 [start-end] 用法;
或者保证自己的程序在 LC_ALL=C 环境下执行~

*如果上游想向后兼容 [a-z]，并希望匹配 "â" 类的字符，字典顺序估计得改成
AÀ .. ZZ`  ->   aâ .. zz`; 目前还没有定论
