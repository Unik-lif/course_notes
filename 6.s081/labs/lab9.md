## 实验记录
这一次的实验似乎前置工作比较多，不过看完了书本之后，感觉看起来实验任务没有那么骇人了。

### Part 1: Large File
看起来这个实验不难，我们主要是需要修改`bmap`的结构，让它可以装下更多的东西。

除了把NDIRECT的值改成11以外，我们主要需要修改两个函数就行，一个是`bmap`，还有一个是`itrunc`。

这个并不困难，只不过太久没写代码了，有点不大细心，用GDB调一下很快就出来了，主要还是照猫画虎着写。

### Part 2: Symbolic links
通过软连接，可以让一个文件名指向对应的inode？

Symbolic links (or soft links) refer to a linked file by pathname; when a symbolic link is opened, the kernel follows the link to the referred file.

主要需要实现的功能是，把一个路径与target所指示的目标所绑定起来，这样之后访问这个path，就会自动陷入到target指示的位置上去。

先前对于文件系统的解读，似乎还不够，需要加上最后这一部分，用来帮助我们完成实验的第二部分。

### 基本解读
首先我们看sys_link这个系统调用，以及sys_unlink系统调用。
```C
// Create the path new as a link to the same inode as old.
uint64
sys_link(void)
{
  char name[DIRSIZ], new[MAXPATH], old[MAXPATH];
  struct inode *dp, *ip;

  if(argstr(0, old, MAXPATH) < 0 || argstr(1, new, MAXPATH) < 0)
    return -1;
  // 首先获取需要之后我们设置link连接的文件
  // 这个文件对应的是old路径
  begin_op();
  if((ip = namei(old)) == 0){
    end_op();
    return -1;
  }
  // 我们通过获取old路径对应的inode pointer来对其内容进行解读
  ilock(ip);
  // 在为文件夹的情况下，我们就不操作了
  if(ip->type == T_DIR){
    iunlockput(ip);
    end_op();
    return -1;
  }
  // 由于要增加连接，所以此处在inode位置增加一个连接
  ip->nlink++;
  // 通过iupdate把更新后的inode pointer信息写到磁盘中的dinode里
  iupdate(ip);
  // iunlock只是释放，并没有干更新之类的事情
  iunlock(ip);
  // 找到父文件夹对应的directory pointer，其实这个也是一个inode
  if((dp = nameiparent(new, name)) == 0)
    goto bad;
  ilock(dp);
  // 在父文件夹中，通过name进行lookup，找到directory pointer底下的一个空的empty dirent entry，把这个文件写进去
  if(dp->dev != ip->dev || dirlink(dp, name, ip->inum) < 0){
    iunlockput(dp);
    goto bad;
  }
  iunlockput(dp);
  iput(ip);

  end_op();

  return 0;

bad:
  ilock(ip);
  ip->nlink--;
  iupdate(ip);
   
  end_op();
  return -1;
}

uint64
sys_unlink(void)
{
  struct inode *ip, *dp;
  struct dirent de;
  char name[DIRSIZ], path[MAXPATH];
  uint off;

  if(argstr(0, path, MAXPATH) < 0)
    return -1;
  // 找到path所对应的父文件夹
  begin_op();
  if((dp = nameiparent(path, name)) == 0){
    end_op();
    return -1;
  }
  // 打开父文件夹对应的directory pointer
  ilock(dp);

  // Cannot unlink "." or "..".
  if(namecmp(name, ".") == 0 || namecmp(name, "..") == 0)
    goto bad;

  // 通过dirlookup检索到父文件夹中文件名为name的inode pointer，并且把offset也获取过来了
  if((ip = dirlookup(dp, name, &off)) == 0)
    goto bad;
  // 现在来控制一下ip，看一下它里头的信息
  ilock(ip);

  if(ip->nlink < 1)
    panic("unlink: nlink < 1");
  // 拒绝接受类型为T_DIR且文件夹不为空这种情况下的请求
  if(ip->type == T_DIR && !isdirempty(ip)){
    iunlockput(ip);
    goto bad;
  }
  // 在directory pointer中，写入一个空的directory entry，这样就能清空了
  memset(&de, 0, sizeof(de));
  if(writei(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
    panic("unlink: writei");
  // 当ip->type为DIR类型，并且为空的
  // 我们删掉ip的同时，那就说明这个文件夹的'..'和'.'已经可以去掉了，而'..'对应的就是dp->nlink
  if(ip->type == T_DIR){
    dp->nlink--;
    iupdate(dp);
  }
  iunlockput(dp);

  ip->nlink--;
  iupdate(ip);
  iunlockput(ip);

  end_op();

  return 0;

bad:
  iunlockput(dp);
  end_op();
  return -1;
}
```
对于这一部分进行研究之后，我们来调研一下create函数的功能。本质上，它是为一个inode创建一个新的名字。
```C
static struct inode*
create(char *path, short type, short major, short minor)
{
  struct inode *ip, *dp;
  char name[DIRSIZ];

  // 根据path，得到directory pointer
  if((dp = nameiparent(path, name)) == 0)
    return 0;

  // 锁定一个directory pointer
  ilock(dp);

  // 查阅directory下的name，得到该name对应的inode pointer
  if((ip = dirlookup(dp, name, 0)) != 0){
    // 如果能够找到的话
    iunlockput(dp);
    ilock(ip);
    // 如果我们要创建的恰好是T_FILE类型，并且当前的ip指向类型是T_FILE或者T_DEVICE中的其中一个
    // 那直接返回就可以了
    if(type == T_FILE && (ip->type == T_FILE || ip->type == T_DEVICE))
      return ip;
    // 如果不是这些类型的，那就寄了
    iunlockput(ip);
    return 0;
  }
  
  // 如果恰好这个name下找不到一个inode pointer
  // 那就分配一个类型指定为type类型的inode pointer
  if((ip = ialloc(dp->dev, type)) == 0)
    panic("create: ialloc");

  // inode pointer获取，并且做一些更新，然后更新到磁盘中去
  ilock(ip);
  ip->major = major;
  ip->minor = minor;
  ip->nlink = 1;
  iupdate(ip);

  // 如果type指向的类型是T_DIR，那还要特定创建一下..和.类型
  if(type == T_DIR){  // Create . and .. entries.
    dp->nlink++;  // for ".."
    iupdate(dp);
    // No ip->nlink++ for ".": avoid cyclic ref count.
    // .. 其实就是自己的父亲吧，.则是自己，因此这边特别加上了这些
    if(dirlink(ip, ".", ip->inum) < 0 || dirlink(ip, "..", dp->inum) < 0)
      panic("create dots");
  }
  // 把这个新文件在父文件夹中连接上
  if(dirlink(dp, name, ip->inum) < 0)
    panic("create: dirlink");
  // 把父文件夹返回
  iunlockput(dp);

  return ip;
}
```
对于create研究之后，我们可以看向其他的系统调用，比如sys_open，其主要内容如下所示：
```C
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  if((n = argstr(0, path, MAXPATH)) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op();

  // 如果加上了CREATE的性质，那么完全可以创建
  // 此外create函数本身考虑了文件已经存在的情况，所以这边调用没有问题
  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    // 检查文件是否存在，也就是inode pointer
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    ilock(ip);
    // 如果是一个文件夹，状态不可以不是只读，那样确实没有办法操作
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }
  // 如果是在DEVICE情况，major号不能太离谱
  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
    iunlockput(ip);
    end_op();
    return -1;
  }
  // 分配一个文件，并为这个文件分配一个file descriptor
  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
    if(f)
      fileclose(f);
    iunlockput(ip);
    end_op();
    return -1;
  }
  // 在这里，根据这个文件inode pointer的类型情况，来撰写一下对应的文件中的具体信息
  if(ip->type == T_DEVICE){
    f->type = FD_DEVICE;
    f->major = ip->major;
  } else {
    f->type = FD_INODE;
    f->off = 0;
  }
  f->ip = ip;
  f->readable = !(omode & O_WRONLY);
  f->writable = (omode & O_WRONLY) || (omode & O_RDWR);

  if((omode & O_TRUNC) && ip->type == T_FILE){
    itrunc(ip);
  }

  iunlock(ip);
  end_op();

  return fd;
}
```
到这里应该解析就已经结束了。