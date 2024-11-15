## Pest的一些基本原则
正则匹配也是类似的，在指定规则的时候，要考虑到最大最长的匹配原则。

一些基本原则：

- Try this.
- If it succeeds, try the next thing.
- Otherwise, try the other thing.
```
(this ~ next_thing) | (other_thing)
```
These rules together make PEGs very pleasant tools for writing a parser.