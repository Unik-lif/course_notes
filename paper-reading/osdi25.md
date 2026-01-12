## keynotes
### Accelerating Software Development
profiler的目的
- 我其实只是要快一点，原本的profiler没法告诉你瓶颈在哪，需要你自己想
- 现在的profiler在AI的加持下，我可以看清楚到底是哪一部分比较慢，然后你可以对这个地方做优化

GDB
- 可以被用于加强，让LLM解释并理解
- 同样也会有安全问题

compiler
- CWhy
- 帮你分析具体是什么错误，不再只是阅读冗长的报错情况

CoverUP
- 利用LLM帮助写测试，LLM读懂对应的定义，然后让LLM对其进行覆盖测试


总结范式：

evolve -> exploit niche -> ensure fitness

exploit niche需要找到一些特性做发散，ensure fitness也许是最重要的，找到不变量来让其更精确工作，也需要更多的事件想清楚