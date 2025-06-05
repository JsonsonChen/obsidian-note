打开 Edge 概率性出现其他应用无法打开
1. 怀疑是应用已经打开只是没有显示，怀疑 BLASTSyncEngine 被堵住导致traslation 无法commite 生效
2. ps -A |grep 进程时没发现该应用进程
3. 进程没创建导致应用没打开
4. 怀疑Zygote 被堵住导致，查看日志发现与 usap 字样
5. usap 功能关闭，依旧还在
6. 查看zygote 进程堆栈发现没什么异常
7. 怀疑socket 被占用，发现zygote socket 正常
8. 出现问题有一个 emmx_zygote 进程占用CPU 高
9. 进一个排查跟zygote 相关的日志，查看ams 进程 发现ZygoteProcess 堵在读取socket 返回中，打印相关日志，发现zygoteServer 有两个，最后发现有个叫AppZygote 的可以让应用自己定义zygote 的功能
10. 没能分析到最终堵住的原因是啥
11. 添加socket读取超时可以解决问题