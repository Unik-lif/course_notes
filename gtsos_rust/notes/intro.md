## Pre:
### Is it Time to Rewrite the operating system in Rust?

C is rightfully called "portable assembly".

In many cases, GC works badly.

Rust: Bowrrowers?

讲的很好，但是语速太快了，我有点跟不上。。

#### 课后习题：
使用rust的前景是在维持C性能的同时有更多的安全考虑，以Rust写OS其实还算一件比较有意义的事。虽然短期做不到推倒重来，这么做的效益也确实很低，但是杂交系统是可以考虑使用的，比如Linus本人也宣布在Linux中添加Rust模块。此外，OS不仅仅是传统的，现在有许多异构操作系统可以进行使用，而虚拟化技术中也有将多台主机虚拟成一个主机的功能（RAID延展），因此与其坐着空想不如好好试试看。
### OS挑战：
1. protability
2. Performance
3. Reliability
4. Usability
5. Security