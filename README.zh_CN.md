# Golang Debugger

## 项目介绍

该项目“**golang debugger**”，是一款面向go语言的调试器，现在业界已经有针对go语言的调试器了，如gdb、dlv等等，那么为什么还要从头再开发一款调试器呢？项目初衷并不是为了开发一款新的调试器，现在上也不是。

我的初衷希望从调试器为切入点，将作者多年以来掌握的知识进行融会贯通，这里的内容涉及go语言本身（类型系统、协程调度）、编译器与调试器的协作（DWARF）、操作系统内核（虚拟内存、任务调度、系统调用、指令patch）以及处理器相关指令等诸多内容。

为什么要从调试器角度入手？
- 调试过程，并不只是调试器的工作，也涉及到到了源码、编译器、链接器、调试信息标准，因此从调试器视角来看，它看到的是一连串的协作过程，可以给开发者更宏观的视角来审视软件开发的位置，也为开发者更全面地认识我国的IT产业提供了一个窗口；
- 调试标准，调试信息格式有多种标准，在了解调试信息标准的过程中，可以更好地理解处理器、操作系统、编程语言等的设计思想，如果能结合开源调试器学习还可以了解、验证某些语言特性的设计实现；
- 调试需要与操作系统交互来实现，调试给了一个更加直接、快速的途径让我们一窥操作系统的工作原理，如任务调度、信号处理、虚拟内存管理等。操作系统离我们那么近但是在认识上离我们又那么远，加强操作系统知识的普及程度对于我们构建更加全面、立体化的IT产业链也有帮助；
- 此外，调试器是每个开发者都接触过的常用工具，我也希望借此机会剖析下调试器的常用功能的设计实现、调试的一些技巧，也领略下调试信息标准制定者的高屋建瓴的设计思想，站在巨人的肩膀上体验标准的美的一面。

简言之，就是希望能从开发一个go语言调试器作为入口切入，帮助初学者快速上手go语言开发，也在循序渐进、拔高过程中慢慢体会操作系统、编译器、调试器、处理器之间的协作过程、加深对计算机系统全局的认识。由于本人水平有限，不可能完全从0开始自研一款调试器，特别是针对go这样一门快速演进中的语言，所以选择了参考开源社区中某些已有的调试器实现gdb、delve作为参考，结合相关规范、标准慢慢钻研的方式。

希望该项目及相关书籍，能顺利完成，也算是我磨练心性、自我救赎的一种方式，最后，如果能对大家确实起到帮助的作用那是再好不过了。

## 阅读本书

1. 克隆项目
```bash
git clone https://github.com/hitzhangjie/golang-debugger-book
```

2. 安装gitbook或gitbook-cli
```bash
# macOS
brew install gitbook-cli

# linux
yum install gitbook-cli
apt install gitbook-cli

# windows
...
```

3. 构建书籍
```bash
cd golang-debugger-book/book

# initialize gitbook plugins
make init 

# build English version
make english

# build Chinese version
make chinese

```

4. 清理临时文件
```bash
make clean
```

> 注意：gitbook-cli存在依赖问题，请尽量使用Node v10.x.
>
> 如果您确实希望使用更新版本的Node，可以通过如下方式来解决:
>
> 1. 如果运行 `gitbook serve` 出错，并且gitbook-cli是全局安装的话，先找到npm全局安装目录并进入该目录，如 `/usr/local/lib/node_modules/gitbook-cli/node_modules/npm/node_modules`, 运行命令 `npm install graceful-fs@latest --save`
> 2. 如果运行 `gitbook install` 出错，进入用户目录下的.gitbook模块安装目录 `.gitbook/versions/3.2.3/node_modules/npm`，运行命令 `npm install graceful-fs@latest --save`

# 意见反馈

联系邮箱 `hit.zhangjie@gmail.com`，标题中请注明来意`golang debugger交流`。

<a rel="license" href="http://creativecommons.org/licenses/by-nd/4.0/deed.zh"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nd/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-nd/4.0/deed.zh">知识共享署名-禁止演绎 4.0 国际许可协议</a>进行许可。

