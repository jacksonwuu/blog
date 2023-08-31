# 如何 Trouble shooting？

Date: 2022.04

调试占据了我工作中大量的时间，在此有必要做一些总结。

Trouble shooting 是解决问题的第一步，从一个非预期的现象出发，发现产生这个现象的本质原因，发现根本原因后才能釜底抽薪，真正解决问题。

我们可以大致遵循这样的流程来 trouble shooting：

1、要对程序的运作有非常清晰的认识。比如说我们做的监控系统，至少要先理解整个系统的数据流向，每一个环节都对数据做了哪些处理，如何对数据做聚合，聚合出来的数据都表达了什么含义等等，要对自己在 trouble shooting 的程序达到非常了解的程度，不然有可能都不会想到出问题的地方。

2、如果有必要，可以把遇到的问题组织成文档，能帮助自己梳理思路，也便于复现问题。

3、通过一些技巧来缩小问题的范围。

-   日志是非常重要的线索，大部分的问题查询日志就可以发现。请重视日志。
-   排除干扰因素，比如说关掉不必要的 services，注释掉没必要的配置等后，再去排查问题就会更清晰。
-   使用正确的工具，包括但不限于监控工具、调试工具、网络工具，比如说 strace 来追踪系统调用的情况、ps 来查看进程运行情况、tcpdump 来抓包等。
-   如果要分析代码执行的问题，可以使用断点/单步跟踪调试，Java 可以在 IDEA 里很方便地断点调试，Go/C/C++ 可以使用 GDB 这样的 debugger。
-   ……

4、当获得初步的错误信息时，比如说一条报错的日志。大部分情况下就知道怎么解决问题了，不知道的话我们还可以去谷歌搜索这条信息，也许之前就有人讨论过这种问题，看看别人是怎么说的。也可以去查文档，很多问题在文档里都有清晰的描述。

5、实在找不到问题所在，就要查看源代码或者请求别人/社区的帮助。

6、问题被发现后要及时用文字总结，这样才能在工作中有所成长。

参考资料：

-   [General Steps to Troubleshoot an Issue](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/introclientissues002.html)
-   [A beginner's guide to network troubleshooting in Linux](https://www.redhat.com/sysadmin/beginners-guide-network-troubleshooting-linux)

-   [Linux 工具快速教程](https://linuxtools-rst.readthedocs.io/zh_CN/latest/index.html)
-   [Linux 常见日志文件](http://c.biancheng.net/view/1097.html)
-   [在 Intellij IDEA 中使用 Debug](https://www.cnblogs.com/chiangchou/p/idea-debug.html)
