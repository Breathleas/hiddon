# hiddon
a undetectiable way to hide rootfs
only a poc, test success on iphonex(ios 13.5.1), odyssey jailbreak

总结一下这段时间对于越狱隐藏的研究:

这个工具主要用于处理rootfs检测, 基于一种非侵入式, 不依赖基板, 不注入任何模块, 不修改目标进程任何代码和数据

原本设想的是通过磁盘系统的mount功能构建一个原始的rootfs,
虽然我们可以直接将原始rootfs快照直接mount出来, 但是必须处理/private/var分区的映射,
但是由于XNU内核不支持mount bind功能(支持mount union但是一个apfs分区不能mount两次), 
所以还是只能修改内核vnode数据实现映射, 也许可以通过mount一个hfs格式的dmg分区实现(没有详细测试)
然后就是mount devfs到我们构建的rootfs上(这一步忘了踩了半天坑), 基本上还是kerbypass那一套

这是第一步, 构建出一个root根分区结构, 第二步就是让目标进程chroot到我们构建的这个rootfs目录
主要有两种方式, 一种是让目标进程调用chroot系统调用进行切换, 第二种就是kernbypass的直接修改proc的rdir和cdir
但是kernbypass是基于基板注入自己的dylib让目标进程暂停下来,  这种带有侵入式的方式本身就很容易被针对性检测

所以我想的是劫持launchd进程的posix_spawn启动子进程, 由于较新版本的ios系统, 苹果应该是对launchd和amfid这两个关键进程,
做了一些特殊防护, 我们自己写的dylib无法注入到launchd进程去,  所以这里采用了exception port来实现拦截.
拦截到posix_spawn之后, 修改attr参数的flags, 加入 POSIX_SPAWN_START_SUSPENDED 标记,
这样目标进程在创建起来之后, 并不会开始执行任何代码, 我们在posix_spawn函数返回后, 可以取得目标进程的pid,
然后使用change root vnode实现chroot即可.  

最开始的时候我测试的是直接修改launchd进程的rdir之后, 就无法启动任何app了, 经过分析发现, 子进程在dyld阶段, 
load system cache的时候失败了, 猜测这应该是posix_spawn的内核实现依赖当前进程的rdir数据, 
只要目标进程创建完毕之后, 在dyld执行之前修改目标进程的rdir, dyld过程并不会受到影响.

经过这段时间对越狱隐藏的研究学习,  基本上也是对越狱过程的一个学习, 这里我只是重点研究了rootfs的隐藏. 
这个poc也是分享给大家一起学习进步,  本身越狱过程对系统的改动是十分巨大的, 越狱隐藏的过程不亚于一次越狱恢复的过程.
所以要在目前公开的越狱工具基础上进行完美的越狱隐藏, 工作量也是非常大的, 而且由于公开的越狱工具并没有开源(完整开源),
导致我们对越狱过程中修改了系统哪些地方,  并不完全所知, 这并不是一个友好的事情. 要想实现完美的越狱隐藏, 最佳的方式还是基于越狱工具本身,
就像android的magisk和magisk hide,  我们需要推动越狱社区和越狱工具往新的方向发展, 越狱工具应当减少对系统的修改, 比如不再挂载rootfs为rw,
而是基于/private/var构建一个虚拟的rootfs给越狱APP使用, 而让部分需要隐藏越狱的app基于原始rootfs运行, 比如对于kernel和deamon的patch不再一刀切,
而是基于调用的进程进行差异化处理, 针对需要隐藏越狱的app执行原始流程逻辑,  这样才能实现深层次的, 不可检测性的越狱隐藏. 



