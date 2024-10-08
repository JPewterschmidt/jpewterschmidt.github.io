---
layout: post
title: 我在个人电脑上使用 Linux
---

**原因**：我全部的学习开发都基于Linux.

## 最初动机

最开始对计算机的深层次应用、技术和原理感兴趣是在高中（职）但
作为一个懒人，虽没能在那个时候就开始系统学习计算机，但也积累了一点常识——
互联网中存在大量C/S架构的应用，S端常常要使用Linux，计算机本科学习应当掌握的四门基础功课分别是什么以及一些基本的数据结构——
这些零碎的常识在我进入本科学习后极大地指导了我的学习。在写这段文字的时候，我仍然到庆幸，自己能较早地意识到这些。

可能是出于个性原因，相比通用的、跨平台的、快速见成果的技术。
我更喜欢晦涩的底层技术，因为我觉得底层的技术可以为我在未来减轻一些竞争，也可以给我对计算机更好地理解。
结合入学前对计算机的粗浅了解，我就开始了C++和POSIX编程的学习之旅。
学习要求我经常使用Linux，为了更快更好地的掌握它，我便开始在我的笔记本上使用它。

## 使用历程

入学之前到大一是在WSL (Windows Subsystem Linux) 上使用 Linux (distro: Ubuntu) 的。
相对直接挑战在物理机上安装使用Linux，使用WSL对我这种笨蛋新手来说是最无痛、无风险的。
那段时间主要学习C++ (当时还不知道有构建系统这回事，还傻傻地手写裸`gcc`命令编译小demo)。
年纪轻（虽说我现在年纪也很轻）的人就是喜欢折腾点“没用的”东西，比如小时候喜欢研究着给电脑装系统（主要为Windows，
误打误撞装过Ubuntu），相信各位读者也有类似的体验。
大概是大二的时候我第一次听说Manjaro这款Linux发行版，网友对其极高的评价，美观的界面，简单易用的包管理器版本以及较新的软件深深地吸引了我。
当天晚上我就决定格式化硬盘，在个人电脑上全新安装 Manjaro.
反复折腾一个星期后，我终于适应了无Windows的学习工作环境，并且艰难地给各种常用软件找了替代品。
强行换用Manjaro后的数月，我逐步了解到了桌面环境、包管理器、Systemd、内核之间的关系。
也大概了解了各发行版之间的关系。从那时起的很长一段时间，KDE都是我的最爱。

在初步使用Manjaro后，我逐步发现了这款发行版以及其社区的一些问题（或者说，我不喜欢的地方）：

- 在对 Archlinux 进行修改后，从 Archlinux 直接继承了大量软件包。
    - 有意无意地，用户往往会被某些非官方资料引导启用 Archlinux 的仓库，导致兼容性问题（日后细说）。
- 使用 GUI 安装 Nvidia 专有驱动时间可能由于对 X11 的配置存在错误，导致无法启动SDDM和KDE。
- 社区中，跟风的以及缺乏基本Linux常识的用户激增。

这些问题也直接导致我放弃 Manjaro，转而选用它的父发行版—— ArchLinux.

ArchLinux 是我唯二最喜欢的发行版，也是到现在（2024年7月）为止使用时间最长的发行版，大概从大二下用到本科毕业。
Arch除了允许我维持一些使用Manjaro期间养成的习惯外，还提供了相对更完整的软件包体系。
Arch还是一款遵从KISS哲学的发行版——十分简约的同时，给予用户相对较大定制自由。这也是吸引我的点。
除此之外，还有优秀的社区建设和完善的文档体系。这些为我日常使用Linux提供了便利。时至今日，我已不使用Arch，但仍经常参考Arch Wiki.
软件包分发方面，Arch采用滚动发行的方式，`pacman` 正常情况下会每次更新会保持系统的所有部分都处于最新状态。
这样就免去了多版本依赖管理问题，极大地简化了软件包管理的难度，用户也无需配置各种第三方仓库，无需仔细管理多版本依赖。
这使得ArchLinux的包管理非常简单，非常适合我这种懒人。

无论是日常用Linux，还是日常写并行代码。都让我加深了对辩证法的理解。
世间一切事物都有矛盾，在计算机的世界也是一样。
Arch 包管理系统的便捷，是拿灵活性换得！系统正常情况下，每个软件包只有一个版本被安装(至少在包管理器看来)
每次更新，都要保持系统所有软件都是最新，这就要求用户经常更新系统。若用户有数月不更新系统，下一次更新时依赖就可能炸掉。
这对学习期间的本科生来说没什么，但对备考的或日常工作比较忙的人就不友好了。
因此，在经过简单的调研后，我决定换用Gentoo Linux.

Gentoo Linux 作为一款 Linux 发行版为用户提供无与伦比的定制自由。
其包管理系统功能非常强大，可以通过一种叫做 USE flags 的机制管理系统和各软件包的特性。
这是 Gentoo 的一大特色，也是我使用它时第一时间注意到的特性。
作为一款 *元发行版* 不同于传统基于二进制发行的发行版，Gentoo 需要用户在自己电脑上编译要使用的软件。
配合 USE flags 机制，用户可以在编译安装时自由添加或删除相关软件特性，
如果用户设置得合理科学，在 Gentoo 上安装的软件包体积可能会比传统软件包小，性能也许会比传统分发的二进制包好。
这是通过避免编译无需功能代码以及在编译时使用特定CPU特性的得来的。
二进制分发的发行版，维护者往往需要考虑不同硬件平台的兼容性问题，不会启用一些较新的指令集等。
而 Gentoo 的包管理系统允许用户通过USE在内的系列机制根据自身的需要和用户机器特性，编译出最合适的软件。
此外，Portage 允许系统中同一软件包的不同版本并存，相比 Pacman 更灵活。
另外，Portage 允许用户长时间不更新系统，而在下一次更新时也不陷入依赖地狱。这点Arch就不行。
