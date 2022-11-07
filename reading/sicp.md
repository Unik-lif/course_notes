## 所以简简单单是一本书的安利
### 书籍名称：SICP
### 课程安利：CS61a
你是否纠结于递归？你是否对wishful-thinking嗤之以鼻或者依旧在观望状态？你是否对于算法复杂度和空间复杂度表示懵懂？你是否好奇开发大型软件的模块与各抽象级之间的联系？以及，你是否想用上古电脑上古语言搓一个解释器轮子？

在人们发现并行时效能可以显著提高前，世界是单核的。外网论坛曾经有个读者暴论，如果有人认为单核还能整出来一些新活，可能他没有看过SICP。

似乎我们学校和学院缺少软件方面的课，SICP或许可以帮帮你看到诡谲的风景。

关于SICP使用的语言scheme，读者虽然可能觉得很老派且单调，以及对"括号神教"感到战战兢兢，但实际上这玩意儿上手奇快，功能强大，是造轮子的很好选择（逃）。阅读SICP原书（最好习惯英文原版）可能存在或多或少的问题，根据本人自己的经历，大家可以在对函数式编程有一个大致的了解再开始撸这本书。

**一个可行的方法**是尝试去学完$cs61a$再来，该课程在**南京大学**也有一个汉化版本，如果英语还没适应过来可以尝试。

此门课程中规中矩，总体上有大量的简单python递归练习和四个非常有趣的项目。比如，第三个项目稍微介绍了OOP的想法并写了一个类似植物大战僵尸的小作业（你真的可以在马原之类的课玩一小会儿），最后一个项目要求读者在给定python框架下写一个scheme解释器。
写scheme解释器难度分为正常版和挑战版，后者可能需要对解释器的细节和总体有一定的把控（我目前没狗胆做），**而前者因为提示很多变的像一个体力活，适合体验流程和假期摆烂（×）。**

关于$cs61a$的课评：
简单和开心的一门课，虽然耗时其实也蛮多的（一般会学一个月附近），没有任何缺点。唯一需要注意的事情是，最好选john的课，他很棒，但是毕竟是资深程序员，他脑子转的很快，某些部分不容易跟上。

做完cs61a后现在你应该有一个scheme解释器了，可以在上面运行Scheme代码了！（可以考虑开始看SICP了）

关于SICP此书本人的阅读建议：**尽可能把习题做完**。当然本人目前也是在缓慢推进这个小小的做题工作。丰富的训练量是必要的，不然大概率和我一开始一样寄了，看了老半天get不到其中的有趣之处。

**强调一遍：建议尽可能做一些习题，把他当华东师大数学分析那本书来做。跳过部分有重要思想的题目，可能会直接导致基础不好的同学（比如我）看不懂后面的章节。**

如果你写过python你可能不会觉得这本书那么惊艳，但是如果你是C语言狂热依赖者可能会有范进中举的感觉。

**分章节阐述：**

#### 第一章
简单介绍了scheme语言，捎带着过了一遍匿名函数lambda和high-order function，最后的习题任务是写一个牛顿法求不动点的方法，想要写微积分函数偷懒的同学可以尝试。似乎不动点还是天坑方向PL中一个很有趣的话题。

#### 第二章
强调各个抽象层的使用和封装，学习数据结构的同学可以体验。不得不说没有for的话对递归的训练是真的充足啊。作者提供了大量的有趣例子，比如如何自己手搓一个小型的形式求导器并给他按上一些琐碎的功能，如何解析基本计算式（这个在后续第四章有大用），如何用不同数据结构管理数据（含部分简单的数据结构内容），以及本人倍感惋惜的图形映射框架玩具（这个玩具因为比较上古，在我coding完后只好proof-reading了，因此本人提供的练习答案可能会有一点点问题）。

本章同样讲述了如何通过添加**tag**的方式对不同模块做好管理，用以防止混淆，并wishful-thinking地采用get和put指令来在一个package里头添加指令，但get和put指令可能要到第三章才会告诉你怎么搭建一个哈希表并使用之。

最后，和Onur Mutlu在其对于抽象层的讲座中类似（Onur Mutlu是做体系结构存储方向的大牛，其一直强调在抽象层的构建和跨越中做一些硬核的工作，比如上至算法，下至device的广义体系结构），作者提出需要尝试在各抽象级间建立联系。
"It is meaningful to define operations that cross the type boundaries"
模块化和抽象化是本章和第三章的核心。
#### 第三章
目前看上去比较诡异，当然我没看到第三章这里，不过在这边你或许会有在写verilog搞电路的感觉。（看到这里再更新吧）
cs61b完成，可以开始施工力！
#### 第四章和第五章
应该是比较精彩的地方，会教你怎么造轮子。

至于学完SICP干什么（×SICP会给你打下比较坚实的基础，你可以尝试去学习haskell和UW的编程语言课）
鄙人愚见：有些东西虽然脱离实用主义（至少现在几乎没人拿scheme编程），但是思想很值得学，没准在哪一天用上了呢（比如Haskell似乎就蛮火的）？即便是现在大火的人工智能方向，感觉最吃香的工作可能还是TVM，OpenBLAS这类东西。而他们的作者，比如陈天奇，是有非常深厚的系统底子的。

诚愿读者乐在其中（√）。
### 优势：
章节短小（指习题部分之间的正文），可以用零碎时间去尝试。
### 劣势：
时间黑洞，而且习题并不是很容易（至少目前我做的挺慢的）。
Tips：
在学习编译原理之前，如果你已经吃了不少SICP的内容，或许会让你的旅途愉快一些，当然本人寄了，因为本人当时没这个想法。
括号有可能会成为你心中的语法糖（暴论）
-----------------------------
2022年4月27日更新
论文整的差不多了，本人已经重新开始施工，不得不说没有for的话对递归的训练是真的充足啊（爽快.jpg）

为了给各位一个直观的对scheme工程的印象（括号神教），请看看下面这个函数，不用看懂，只要做好一定的心理准备：
```scheme
(define (balanced-mobile mobile)
    (cond ((and (not (is-mobile? (l-structure mobile))) (not (is-mobile? (r-structure mobile))))
            (and (= (* (l-length mobile) (l-structure mobile)) (* (r-length mobile) (r-structure mobile)))))
          ((and (not (is-mobile? (l-structure mobile))) (is-mobile? (r-structure mobile)))
            (and (= (* (l-length mobile) (l-structure mobile)) (* (r-length mobile) (total-weight (r-structure mobile)))) (balanced-mobile (r-structure mobile))))
          ((and (is-mobile? (l-structure mobile)) (not (is-mobile? (r-structure mobile))))
            (and (= (* (l-length mobile) (total-weight (l-structure mobile))) (* (r-length mobile) (r-structure mobile))) (balanced-mobile (l-structure mobile))))
          ((and (is-mobile? (l-structure mobile)) (is-mobile? (r-structure mobile)))
            (and (= (* (l-length mobile) (total-weight (l-structure mobile))) (* (r-length mobile) (total-weight (r-structure mobile)))) (balanced-mobile (l-structure mobile)) (balanced-mobile (r-structure))))    
    )
)
```

最后送给读者一个很有意思的小玩意儿，请配合视频对本书进行阅读，来自古神关于SICP的注解：

（视频本家：魔理沙偷走了重要的东西！）
https://www.bilibili.com/video/BV1ax411N7FL?spm_id_from=333.337.search-card.all.click

![令人非常可惜的是，本视频仅仅包含了第一章和第二章的琐碎内容，如果读者只能坚持看到视频所述的部分内容，或者仅仅刷题到car，cdr，cddr之类的部分，或许就是入宝山而两手空空而归吧（自裁请）。

![Image description](https://kernel-cdn.niconi.org/2022-05-10/1652195878-41432-screenshot-2022-05-10-231553.png)


**来自2022年5月底的绝望回复：**

嘴硬了，真心感觉死做题没什么意义，比如第二章九十多道题我老老实实写到八十多，扫一眼后已经感觉后面的题目没有太大必要去做了，而且确实无法忍受只能proof-reading而无法运行的绝望感觉，各位还是各取所需吧。
（麻蛋读了这么多年还是没有摆脱学生思维，想把学习当做有量化指标的工程是没有太大意义的）