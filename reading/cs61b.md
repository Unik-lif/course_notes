因SICP书中涉及cs61abc中的部分思想，同样为了避免开新帖的一些麻烦，简单在此记录cs61b体验如下（之后打算开坑的cs61c亦如是）。
## sp18 cs61b：

#### 版本：
cs61b存在多个版本，如北大信科飞猪所述，由Josh Hug讲述的sp18版本是测试集和练习量最完全的。

其余的版本也有很棒的参考，比如带有著名项目GitLet的sp21版本，以及每年暑假都会开大致一个月，主要基于实验且短期workload巨大的暑期版本cs61bl，今年的已经结束了，不过同学们可以参考2020年的cs61bl，现在这个版本还是开放的。

除了公开课以外，部分往年视频资料可能需要伯克利的邮箱账号，这玩意其实花点小钱钱不难搞到。

本人选用的版本是sp18。

#### 前提：
cs61b是伯克利计算机的第二门课，其与cs61a有一定关联性但是并不大。和cs61a的小打小闹不同，该课程项目的代码量还是比较可观的（一个项目上千行还是能够保证的，当然如果你水平高不需要很多private helper函数封装当我没说），因此本人推荐后来者最好腾出一段可能没有那么忙碌的时间，给自己大量的空闲时间去执行这件小任务。

该课程的弹性时间很大，笔者感觉自己可能已经花了大致150小时了（包括看34小时左右的视频），因此希望大家量力而行。
#### 我能学到什么？
1. 基本的Java语法，static private public等关键字所蕴含的OOP开发思想，TDD（Test Driven development）思想。
2. 数据结构的设计理念，涉及链表，树，队列，栈，递归，图，各种排序等诸多算法，以及动态规划初探（该课程中了解即可）。
3. 开发比较大的程序的一些理念和想法（如何化整为零，如何规划好数据结构）。

#### 我不能学到什么？
比较高级的Java特性，比如线程池，锁，JVM，GC之类的，那种东西比较难，这个课暂且不涉及。

#### 印象比较深的经历：
##### 项目：
随机生成地牢迷宫的算法并设计自己的地牢DDF游戏，该项目有很高的自由度（当然也会稍微难一些）。
构造行星运动轨迹的NBody项目，简单有趣。
利用A*算法搭地图的项目，这个项目不难，其中地图栅格化的思想很值得学习和参考。在处理海量数据中（总之不小了，大概有39个MB吧）真切感到数据结构和算法的强大。
最朴实无华的写测试集和ArrayList数据结构的小项目。（其实好像也没那么容易）
##### 部分作业：
Percolation渗露法模拟。
修改电吉他音色以实现不同效果。
A*算法初探。
##### 课程：
slides，video，guide，MidTerm全面发力，认真做的话习题训练量巨大
比较贴心的Discussion，可以即时考验知识的掌握能力。
##### 实验：
把涉及的数据结构和算法大部分都实现了一遍，也蛮好的。

##### 搞怪： 
https://www.bilibili.com/video/BV1qt411W7dh?p=190&vd_source=49f5b184846e52adee8a1a5165c5f962
特别好玩的有声音的排序动画（比如冒泡排序听起来真的和冒泡一模一样），本节课的槽点从第42分钟开始集中爆发，读者就自己去看看吧，还是卖个关子比较好。
https://www.bilibili.com/video/BV1qt411W7dh?p=197&spm_id_from=pageDriver&vd_source=49f5b184846e52adee8a1a5165c5f962
第十五分钟名震中外的我爸是李刚。
##### 一些别的：
1. Josh Hug曾任职于普林斯顿，所以课程涉及了很多普林斯顿红色算法宝书的内容。在讲课变难的时候可以看书，这本书写的真的很好。
2. 该课的后续课程为cs170或者cs188。
3. 分享一个蛋用没有的桶排序的MSD递归，自己写出来感觉还是蛮好玩的。这应该是最典型的华而不实吧。
```
 sortHelperMSD(...){
        ...
        for (int i = 0; i < radix[i]; i++) {
            sortHelperMSD(asciis, radix[i], radix[i + 1], index + 1);
        }
}
```
（诚愿体验者乐在其中）