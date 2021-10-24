## 4.1 目的

尽管开发者花费了大量精力来避免自己的代码中引入bug，但是写出bug仍然是一个很平常的事情。开发人员希望定位代码中的问题时，通常会借助 **Print语句**（如`fmt.Println`） 来打印变量的值，进而推断程序执行结果是否符合预期。在某些更复杂的场景下，打印语句可能难以胜任，调试器会更好地协助我们定位问题。

调试器可以帮助我们控制tracee（被调试进程、线程）的执行，也可以观察tracee的运行时内存、寄存器状态，借此我们可以实现代码的逐语句执行、控制代码执行流程、检查变量值是否符合预期，等等。

我认为调试器对于刚入门的开发者而言，是一个不可缺少的工具，它还能加深对编程语言、内存模型、操作系统的认识。即便是对于一些从业多年的开发者而言，调试器也会是一个有用的帮手。

本书将指导我们如何开发一个面向go语言的调试器，如果读者之前有使用符号级调试器（如gdb、delve等）的经验，那对于理解本书内容将会非常有帮助。

调试器要支持的重要操作，通常包括：

- 设置断点，在指定内存地址、函数、语句、文件和行号处设置断点；
- 单步执行，单步执行一条指令, 单步执行一条语句, or 一直运行到下个断点处；
- 获取、设置寄存器信息；
- 获取、设置内存信息；
- 对表达式进行估值计算；
- 调用函数；
- 其他；

本书后续章节会介绍如何实现上述操作，如果对调试器内部工作原理好奇，那就请继续吧。

 