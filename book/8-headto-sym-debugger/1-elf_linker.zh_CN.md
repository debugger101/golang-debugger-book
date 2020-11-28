## 链接器 & 加载器

### 链接器链接动作

链接，就是链接器将多个可重定位文件（目标文件、库文件）进行符号解析、重定位、将多个文件结合成一个可执行程序的过程。

- 静态链接，指的是，编译时执行链接，链接的是静态库；

- 动态链接，指的是，加载时或运行时执行链接，链接的是动态库；

### 加载器加载动作

加载，就是程序运行时，链接器将程序数据及其依赖的库文件加载到内存的过程，以允许程序正常执行。

![](assets/elf_loader.png)

- 静态加载，指的是，在程序启动前加载过程中，执行可执行程序、库的内存地址映射；

- 动态加载：指的是，在进程启动之后，执行库的内存地址映射；

### 链接器&加载器协作

可以肯定的是，链接器和加载器之间不是完全孤立的，它们之间也是一种协作关系。

- 静态链接，静态加载。链接器/usr/bin/ld使用静态库(.a)链接，加载器是内核本身；

- 静态加载，动态链接。链接器/usr/bin/ld使用动态库(.so)链接，加载器是二进制解释器，如在debian9上是/lib64/ld-linux-x86-64.so.2（该路径现在对应的是/lib/x86_64-linux-gnu/ld-2.24.so），这个解释器对应so文件的加载由内核完成，可执行程序本身也由内核完成；

- 静态链接，动态加载。linux上没有使用；

- 动态链接，动态加载。加载器是通过libdl库进行dlopen进行加载，链接器的工作分散在libdl和用户程序中，如dlopen加载库，dlsym解析库就涉及到动态链接。

注意到当链接器使用静态链接或动态链接时，加载器的执行逻辑有所变化。静态链接时内核自己来加载可执行程序和库，动态链接时则将库的加载动作交给二进制解释器来代为处理。

那内核是如何来识别这种差异的呢？这里的桥梁就是ELF文件中的信息，它表明了某些某些库信息在链接时是使用了何种链接方式，内核据此作出不同处理。



### 参考文献

1. What are the executable ELF files respectively for static linker, dynamic linker, loader and dynamic loader, Roel Van de Paar, https://www.youtube.com/watch?v=yzI-78zy4HQ