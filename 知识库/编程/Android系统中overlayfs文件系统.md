【Android 动态分区详解(七) overlayfs 与 adb remount 操作 - CSDN App】https://blog.csdn.net/guyongqiangx/article/details/128881282?sharetype=blog&shareId=128881282&sharerefer=APP&sharesource=weixin_42536846&sharefrom=link

本文详细介绍了Android系统中overlayfs文件系统的原理及其在remount操作中的应用，特别是指出了overlayfs在A/B系统分区管理和调试过程中的重要性。具体如下：

1. OverlayFS是一种基于其他文件系统之上的堆叠式文件系统，常用于合并不同目录的内容并向用户提供统一视图。
2. 在Linux系统中，overlayFS的相关代码位于fs/overlayfs目录，而在Android系统中则增加了额外的功能和限制条件。
3. Android利用overlayFS实现在只读分区上进行可逆的修改，开发者可通过reboot使更改持久化，或者仅针对特定分区启用overlayFS。
4. 使用adb工具结合disable-verity和remount命令可以在Android设备上方便地管理overlayFS挂载以及相关的文件系统操作。