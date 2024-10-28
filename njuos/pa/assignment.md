## PA0
### VIM usage
```
i1<ESC>q1yyp<C-a>q98@1
```
q1是记录这个名字叫1的宏，98@1是结束这个名为1的宏。

yyp是复制当前行，并向下粘贴，Ctrl A在Vim中，一般用于对光标所在的位置的数字进行递增操作。

下面的98是执行的次数。之后就重复之前的向下复制，并且递增的过程，执行98次，因此最后我们可以得到1-100这样的效果。

```
<C-v>24l4jd$p
```

### Unix哲学
这种东西确实好：
```
find . -name "*.[ch]" | xargs grep "#include" | sort | uniq
```

一些工具：
```
熟悉一些常用的命令行工具, 并强迫自己在日常操作中使用它们
文件管理 - cd, pwd, mkdir, rmdir, ls, cp, rm, mv, tar
文件检索 - cat, more, less, head, tail, file, find
输入输出控制 - 重定向, 管道, tee, xargs
文本处理 - vim, grep, awk, sed, sort, wc, uniq, cut, tr
正则表达式
系统监控 - jobs, ps, top, kill, free, dmesg, lsof
上述工具覆盖了程序员绝大部分的需求
可以先从简单的尝试开始, 用得多就记住了, 记不住就man
```
### git
对于git我目前仅仅知道一些粗暴的使用方法，应该抽空学习一下这些东西：

https://onlywei.github.io/explain-git-with-d3/

git commit --allow-empty这个很有意思

**PA0 Done**