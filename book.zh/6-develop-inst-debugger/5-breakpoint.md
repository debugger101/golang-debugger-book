## 动态断点

### 实现目标：添加断点

断点按照其生命周期进行分类，可以分为静态断点和动态断点。

-   静态断点的生命周期是整个程序执行期间，一般是通过`int 0x3h`来强制插入`0xCC`充当断点，其实现简单，在编码时就可以安插断点，但是不灵活；
-   动态断点的生成、移除，是通过运行时指令patch，其生命周期是与调试活动中的操作相关的，其最大的特点就是灵活，一般是只能借由调试器来生成。

不管是静态断点还是动态断点，其原理是类似的，都是通过一字节指令`0xCC`来实现暂停任务执行的操作，处理器执行完`0xCC`之后会暂停当前任务执行。

>   ps：我们在章节4.2中有提到`int 0x3h`（编码后指令0xCC)是如何工作的，如读者忘却了其工作原理，可以查阅相关章节。

断点按照实现方式的不同，也可以细分为软件断点和硬件断点。

-   硬件断点一般是借助硬件特有的调试端口来实现，如将感兴趣的指令地址写入调试端口（寄存器），当PC命中时就会触发停止tracee执行的操作，并通知tracer；
-   软件断点是相对于硬件断点而言的，不借助于硬件特有调试端口来实现的，一般都可以归为软件断点。

我们先只关注软件断点，并且只关注动态断点。断点的添加、移除是调试过程的基石，在我们掌握了在特定地址处添加、移除断点之后，我们可以研究下断点的应用，如step、continue等。

在熟练掌握了这些操作之后，我们将在下一章结合DWARF来实现符号级断点，将允许你对一行语句、函数、分支控制添加、移除断点，届时断点的作用就进一步凸显出来了。

### 代码实现

我们使用`break`命令来添加断点，可以简单缩写成`b`，使用方式如下：

```bash
# 注意<locspec>的写法
break <locspec>
```

locspec表示一个代码中的位置，可以是指令地址，也可以是一个源文件中的位置。如果是后者，我们需要查询行号表先将源码中的位置转换成指令地址。有了指令地址之后，我们就可以对该地址处的指令数据进行patch以达到添加、移除断点的目的。

本章节，我们先只考虑locspec为指令地址的情况。

>   TODO: 更详细的locspec解析方式，可以参考dlv中的实现：https://sourcegraph.com/github.com/go-delve/delve@master/-/blob/pkg/locspec/locations.go

现在来看下我们的实现代码：

```go
package debug

import (
	"errors"
	"fmt"
	"strconv"
	"strings"
	"syscall"

	"github.com/spf13/cobra"
)

var breakCmd = &cobra.Command{
	Use:   "break <locspec>",
	Short: "在源码中添加断点",
	Long: `在源码中添加断点，源码位置可以通过locspec格式指定。

当前支持的locspec格式，包括两种:
- 指令地址
- [文件名:]行号
- [文件名:]函数名`,
	Aliases: []string{"b", "breakpoint"},
	Annotations: map[string]string{
		cmdGroupKey: cmdGroupBreakpoints,
	},
	RunE: func(cmd *cobra.Command, args []string) error {
		fmt.Printf("break %s\n", strings.Join(args, " "))

		if len(args) != 1 {
			return errors.New("参数错误")
		}

		locStr := args[0]
		addr, err := strconv.ParseUint(locStr, 0, 64)
		if err != nil {
			return fmt.Errorf("invalid locspec: %v", err)
		}

		orig := [1]byte{}
		n, err := syscall.PtracePeekData(TraceePID, uintptr(addr), orig[:])
		if err != nil || n != 1 {
			return fmt.Errorf("peek text, %d bytes, error: %v", n, err)
		}
		breakpointsOrigDat[uintptr(addr)] = orig[0]

		n, err = syscall.PtracePokeText(TraceePID, uintptr(addr), []byte{0xCC})
		if err != nil || n != 1 {
			return fmt.Errorf("poke text, %d bytes, error: %v", n, err)
		}
		fmt.Printf("添加断点成功\n")
		return nil
	},
}

func init() {
	debugRootCmd.AddCommand(breakCmd)
}
```

这里的实现逻辑并不复杂，我们来看下。

首先假定用户输入的是一个指令地址，这个地址可以通过disass查看反汇编时获得。我们先尝试将这个指令地址字符串转换成uint64数值，如果失败则认为这是一个非法的地址。

如果地址有效，则尝试通过系统调用`syscall.PtracePeekData(pid, addr, buf)`来尝试读取指令地址处开始的一字节数据，这个数据是汇编指令编码后的第1字节的数据，我们需要将其cache起来，然后再通过`syscall.PtracePokeData(pid, addr, buf)`写入指令`0xCC`。

等我们准备结束调试会话时，或者显示执行`clear`清除断点时，需要将数据这里的0xCC重新替换会cache的原始数据，

### 代码测试

下面来测试一下，首先我们启动一个测试程序，获取其pid，这个程序最好一直死循环不退出，方便我们测试。

然后我们先执行`godbg attach <pid>`准备开始调试，调试会话启动后，我们执行disass反汇编命令查看汇编指令级对应的指令地址。

```bash
godbg attach 479
process 479 attached succ
process 479 stopped: true
godbg> 
godbg> disass
.............
0x465326 MOV [RSP+Reg(0)+0x8], RSI
0x46532b MOV [RSP+Reg(0)+0x10], RBX
0x465330 CALL .-400789
0x465335 MOVZX ECX, [RSP+Reg(0)+0x18]
0x46533a MOV RAX, [RSP+Reg(0)+0x38]
0x46533f MOV RDX, [RSP+Reg(0)+0x30]
.............
godbg> 
```

随机选择一条汇编指令的地址，在调试会话中输入`break <address>`，我们看到提示断点添加成功了。

```bash
godbg> b 0x46532b
break 0x46532b
添加断点成功
godbg>
godbg> exit
```

最后执行exit退出调试。

这里我们只展示了断点的添加逻辑，断点的移除逻辑，其实实现过程非常相似，我们将在clear命令的实现时再予以介绍。

实现了break、clear之后我们再来看step、continue等控制执行流程的调试命令的实现。



