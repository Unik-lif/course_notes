记录一下自己在写程序的时候犯的一些令人啼笑皆非的低级错误，常看常新，要让自己刺痛并且难过，那就对了！

有时候最佳实践并不容易找，我以前确实有一些最佳实践遇到过，比如：
- 0 == i，把常数写到前面，从而避免等号只有一个的问题

所以没有办法，最佳实践有时候就是没用，我也不能指望别人看我的代码帮忙。

我在过去的一些编程活动中经历了非常多的逆天错误，不一定是我自己找出来的，可能是对照别人代码找到的。

简单记录如下：

### MIT 6.s081
> 在 Lab8 中把条件判断写到了 for loop 里，导致程序还没有遍历完全部的 CPU ，就提前退出了。

#### 问题描述:
```C
    for(int i = 0; i < NCPU && i != id; i++){
      acquire(&kmem[i].lock);
      r = kmem[i].freelist;
      if(r){
        kmem[i].freelist = r->next;
        
        // kmem[id].freelist = r;
        release(&kmem[i].lock);
        break;
      }
      release(&kmem[i].lock);
    }
  
```
造成的结果是，我一直以为是同步和锁之间的事情没有做好，但是仔细想想确实不应该，后来发现有人做法和我一样，就是没有把条件判断写到 loop 里头。

额外浪费了很多精力和心情，也是我开这个坑的原因。他妈的！！
#### 解决方案
避免在 for loop 中写条件判断，其余的条件判断还是以 if 来写，这样不会有歧义，并且会非常清晰。