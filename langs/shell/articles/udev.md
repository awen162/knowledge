# udev

```txt
Jack：淫龙，Linux实现的设备管理机制是什么样子的呢？

我：在2.4内核里，主流的解决方案是devfs。
Jack：我知道。在2.6里，devfs已经被udev替代了。

我：这种说法是不准确的，是一种外行看热闹的说法。
Jack：怎么说？

我：让我给你讲一讲proc文件系统的起源吧。听完了，你自然就明白了。
Jack：proc文件系统？穿越了。

我：在很久很久很久以前，Linux内核的所有代码都是写死的，如果你想修改其中一些参数，必须要手动修改源代码，然后重新编译，重新写软盘，重新跑起来。在很久很久以前，在Unix的/dev目录下出现了一个文件——/dev/mem。这是一个特殊（变异）的设备文件，通过对这个文件的读写（从用户进程），可以改变整个内核的内存。从此，内核的内存可以动态进行修改，而不需要停机修改内核代码、重新编译、重新写软盘、重新启动这一些列的复杂过程。
Jack：然后，然后，王子和公主性福地生活在一起了？
我：去死吧。Linux内核的开发者从/dev/mem这个变异的设备文件中尝到了甜头后，就一发不可收拾了。他们接着在/下设立了一个/proc目录，同时创建了proc文件系统。/proc目录下的文件都是变异型文件，不存在磁盘中，而是在访问时动态建立，存在于ramfs里。这些文件在早期主要是对内存使用情况的统计以及动态修改，而到了后来，内核设计人员的胃口越来越难以满足。所有他们认为易于管理的理念都应该加入这个目录，/proc下就不再局限于内存信息，甚至包括外部设备管理信息——sysfs。
Jack：那这个devfs是什么关系呢？

我：devfs就是在这种背景下成长起来的。算是proc文件系统的一种推广或者变异。但是，devfs有很多致命的缺陷，导致devfs的作者没有解决的能力与信心，终于，决定宣布失败，彻底放弃。然后，在Linux2.5的内核中，尝试构建一种sysfs文件系统。这个文件系统挂接在/sys下。

Jack：devfs的致命缺陷是什么呢？

我：这个问题网上已经很多了，随便Google一下就行了。
Jack：那么udev又是怎么回事儿呢？

我：先不要急。让我把sysfs说完，你再听udev就完全明白了。sysfs相比于devfs，它已经是一套成型的、统一的设备管理模型。sysfs的描述能力不仅仅局限于干活的外部设备，甚至包括外部总线（pci总线等）。挂接在/sys目录下的文件的根目录是按类型（而非层级）来分类的。包括/sys/bus，/sys/block，/sys/class等。
Jack：那么，udev是怎么来的呢？

我：sysfs文件系统能直观地反应加入计算机的总线、设备的信息（通过kobject对象实现）。但是，它仅仅是一种体现和反应，并不能如devfs一样，在生成设备文件时（不管设备在不在），就直接把设备驱动加载进来（其实，这是不必要的）。如果想智能地控制外部设备的管理，还需要一个任务，一个专业的工具。在devfs里，干这个活儿的是一个内核线程。而udev正好相反，它通过等待内核注册过的设备热插拔引起的事件，激发相应的功能。

Jack：这样有什么优势吗？

我：这样做，最大的优势在于，内核的负载降低了，工作移交到了用户进程里。
Jack：可是，从CPU的角度来讲，不是一样要干活吗？整体的负载并没有降低。
我：是的。从CPU的角度讲，负载并没有降低，但是，devfs所负责的工作并非操作系统最核心的工作，这种工作能移交到用户空间是最好。好处至少有三点：

1、如果该线程（或进程）崩溃，对整个系统而言，在用户空间崩溃比在内核空间崩溃的负面影响要小得多（尤其是在一些金融系统内）。

2、运行在内核空间的线程会碰到大量的同步与互斥的问题，设计起来会比较困难，如果以后想在内核添加一个什么功能（尤其是核心功能），那么难度将会成指数增长。
3、内核并非是垃圾场，不能什么都往里扔，即便现在没有坏。但是，如果根基打坏了，10年后，整个系统遗留下来的问题将会非常可怕。这种现象在各大互联网公司非常严重。
Jack：这么说来，udev和devfs根本就不是同一个层次的概念。devfs不仅包括了一个文件系统机制，还包括了监控外部设备以及干活的线程。而udev仅仅是根据sysfs来干活的一个工具而已。

我：可以这样理解。所以，我说“devfs被udev替代”是一种外行看热闹的说法。
Jack：那你可以说一说devfs与udev实现上的差异吗？
我：不可以。devfs已经在2.6内核里被彻底废弃了，我对它的实现没有认识。关于sysfs+udev可以简单谈一下它的原理。最重要的是3个结构体：kobjects、ksets、subsystems。其中，当一个kobjects结构体对象被生成的时候（驱动工程师实现）会发生两个重要的事情：1、在/sys/目录下会有相应的文件夹。2、产生一个hotplug事件，当有外部设备被插/拔时，内核将会把这个时间主动通知到用户空间的进程（udev。本质上，hotplug是一种信号或者软中断机制，信号也是广义上的软中断）。相比于kobjects的高度抽象，kset更具体一些。具体到什么程度呢？所有具有相同属性的kobjects被分为一个kset。至于什么样的设备具有相同的kobjects属性，由驱动工程是自己判断。而一个设备具体是怎么加入Linux的，就不是设备模型应该考虑的问题，而是驱动工程师的实现。设备模型仅仅是给出了一个框架，所有的设备都需要遵守，但并不是手把手教驱动工程师写驱动
```