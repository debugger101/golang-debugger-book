## 启动&Attach进程

### 实现目标：启动进程并attach

#### 思考：如何让进程刚启动就停止？

前面我们介绍了如何通过`exec.Command(prog, args...)`启动一个进程，也介绍了如何通过ptrace系统调用attach到一个运行中的进程。

一个运行中的进程被attached时，其正在运行的指令可能已经越过了我们关心的位置。比如，我们想通过调试追踪下golang程序在执行main.main之前的初始化步骤，但是通过先启动程序再attach的方式无疑就太滞后了，main.main可能早已经开始执行，甚至程序都已经执行结束了。

考虑到这，不禁要思索在“启动进程”小节的实现方式有没有问题。我们如何让进程在启动之后立即停下来等待被调试器调试呢？如果做不到这点，就很难做到理想的调试。

#### 内核：启动进程时内核做了什么？

启动一个指定的进程归根究底是fork+exec的组合：
```go
cmd := exec.Command(prog, args...)
cmd.Run()
```

- cmd.Run()首先通过fork创建一个子进程；
- 然后子进程再通过execve函数加载目标程序、运行；

但是如果只是这样的话，程序会立即执行，可能根本不会给我们预留调试的机会，甚至我们都来不及attach到进程添加断点，程序就执行结束了。

我们需要在cmd对应的目标程序指令在开始执行之前就立即停下来！要做到这一点，就要依靠ptrace操作PTRACE_TRACEME。

#### 内核：PTRACE_TRACEME到底做了什么？

先使用c语言写个程序来简单说明下这一过程，为什么用c，因为接下来我们要看些内核代码，加深对PTRACE_TRACEME操作以及进程启动过程的理解，当然这些代码也是c语言实现的。

```c
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h>   /* For constants ORIG_EAX etc */
int main()
{   pid_t child;
    long orig_eax;
    child = fork();
    if(child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", NULL);
    }
    else {
        wait(NULL);
        orig_eax = ptrace(PTRACE_PEEKUSER, child, 4 * ORIG_EAX, NULL);
        printf("The child made a system call %ld\n", orig_eax);
        ptrace(PTRACE_CONT, child, NULL, NULL);
    }
    return 0;
}
```

上述示例中，首先进程执行一次fork，fork返回值为0表示当前是子进程，子进程中执行一次`ptrace(PTRACE_TRACEME,...)`操作，让内核代为做点事情。

我们再来看下内核到底做了什么，下面是ptrace的定义，代码中省略了无关部分，如果ptrace request为PTRACE_TRACEME，内核将更新当前进程`task_struct* current`的调试信息标记位`current->ptrace = PT_PTRACED`。

**file: /kernel/ptrace.c**

```c
// ptrace系统调用实现
SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr,
		unsigned long, data)
{
	...

	if (request == PTRACE_TRACEME) {
		ret = ptrace_traceme();
		...
		goto out;
	}
	...
        
 out:
	return ret;
}

/**
 * ptrace_traceme是对ptrace(PTRACE_PTRACEME,...)的一个简易包装函数，
 * 它执行检查并设置进程标识位PT_PTRACED.
 */
static int ptrace_traceme(void)
{
	...
	/* Are we already being traced? */
	if (!current->ptrace) {
		...
		if (!ret && !(current->real_parent->flags & PF_EXITING)) {
			current->ptrace = PT_PTRACED;
			...
		}
	}
	...
	return ret;
}
```

#### 内核：PTRACE_TRACEME对execve影响？

c语言库函数中，常见的exec族函数包括execl、execlp、execle、execv、execvp、execvpe，这些都是由系统调用execve实现的。

系统调用execve的代码执行路径大致包括：

```c
-> sys_execve
-> do_execve
-> do_execveat_common
```

函数do_execveat_common的代码执行路径大致包括下面列出这些，其作用是将当前进程的代码段、数据段、初始化未初始化数据通通用新加载的程序替换掉，然后执行新程序。

```c
-> retval = bprm_mm_init(bprm);
-> retval = prepare_binprm(bprm);
-> retval = copy_strings_kernel(1, &bprm->filename, bprm);
-> retval = copy_strings(bprm->envc, envp, bprm);
-> retval = exec_binprm(bprm);
-> retval = copy_strings(bprm->argc, argv, bprm);
```

这里牵扯到的代码量比较多，我们重点关注一下上述过程中`exec_binprm(bprm)`，这是执行程序的部分逻辑。

**file: fs/exec.c**

```c
static int exec_binprm(struct linux_binprm *bprm)
{
	pid_t old_pid, old_vpid;
	int ret;

	/* Need to fetch pid before load_binary changes it */
	old_pid = current->pid;
	rcu_read_lock();
	old_vpid = task_pid_nr_ns(current, task_active_pid_ns(current->parent));
	rcu_read_unlock();

	ret = search_binary_handler(bprm);
	if (ret >= 0) {
		audit_bprm(bprm);
		trace_sched_process_exec(current, old_pid, bprm);
		ptrace_event(PTRACE_EVENT_EXEC, old_vpid);
		proc_exec_connector(current);
	}

	return ret;
}
```

这里`exec_binprm(bprm)`内部调用了`ptrace_event(PTRACE_EVENT_EXEC, message)`，后者将对进程ptrace状态进行检查，一旦发现进程ptrace标记位设置了PT_PTRACED，内核将给进程发送一个SIGTRAP信号，由此转入SIGTRAP的信号处理逻辑。

**file: include/linux/ptrace.h**

```c
/**
 * ptrace_event - possibly stop for a ptrace event notification
 * @event:	%PTRACE_EVENT_* value to report
 * @message:	value for %PTRACE_GETEVENTMSG to return
 *
 * Check whether @event is enabled and, if so, report @event and @message
 * to the ptrace parent.
 *
 * Called without locks.
 */
static inline void ptrace_event(int event, unsigned long message)
{
	if (unlikely(ptrace_event_enabled(current, event))) {
		current->ptrace_message = message;
		ptrace_notify((event << 8) | SIGTRAP);
	} else if (event == PTRACE_EVENT_EXEC) {
		/* legacy EXEC report via SIGTRAP */
		if ((current->ptrace & (PT_PTRACED|PT_SEIZED)) == PT_PTRACED)
			send_sig(SIGTRAP, current, 0);
	}
}
```

在Linux下面，SIGTRAP信号将使得进程暂停执行，并向父进程通知自身的状态变化，父进程通过wait系统调用来获取子进程状态的变化信息。

父进程也可通过`ptrace(PTRACE_COND, pid, ...)`操作来恢复子进程执行，使其执行execve加载的新程序。

#### Put it Together

现在，我们结合上述示例，再来回顾一下整个过程、理顺一下。

首先，父进程调用fork、子进程创建成功之后是处于就绪态的，是可以运行的。

然后，子进程先执行`ptrace(PTRACE_TRACEME, ...)`告诉内核“**该进程希望在后续execve执行新程序时停下来等待tracer attach**”。子进程再执行execve加载新程序，重新初始化进程执行所需要的代码段、数据段等等。

重新初始化完成之前内核会将进程状态调整为“UnInterruptible Wait”阻止其被调度、响应外部信号，完成之后，再将其调整为“Interruptible Wait”，即可以被信号唤醒，意味着如果有信号到达，则允许进程对信号进行处理。

接下来，如果没有该标记位，子进程状态将被更新为可运行等待下次调度。当内核发现这个子进程ptrace标记位为PT_PTRACED时，则会执行这样的逻辑，内核给这个子进程发送了一个SIGTRAP信号，该信号将被追加到进程的pending信号队列中，并尝试唤醒该进程，当内核任务调度器调度到该进程时，发现其有pending信号到达，将执行SIGTRAP的信号处理逻辑，只不过SIGTRAP比较特殊是内核代为处理。

SIGTRAP信号处理具体做什么呢？它会暂停目标进程的执行，并向父进程通知自己的状态变化。此时父进程通过系统调用wait就可以获取到子进程状态变化的情况。

### 代码实现

go标准库里面只有一个ForkExec函数可用，并不能直接写fork+exec的方式，但是呢，go标准库提供了另一种用起来更加友好的思路。

我们首先通过`cmd := exec.Command(prog, args...)`获取一个cmd对象，接着通过cmd对象获取进程Process结构体，然后修改其内部状态为ptrace即可。这样之后再启动子进程`cmd.Start()`，然后调用`Wait`函数来获取子进程的状态，等子进程停下来，然后父进程可以先做一些调试工作。

这里的示例代码，我们在以前示例代码基础上修改，修改后代码如下：

```go
package cmd

import (
	"errors"
	"fmt"
	"os"
	"os/exec"
	"strings"
	"syscall"

	"godbg/cmd/debug"

	"github.com/spf13/cobra"
)

var pid int

// execCmd represents the exec command
var execCmd = &cobra.Command{
	Use:   "exec <prog>",
	Short: "调试可执行程序",
	Long:  `调试可执行程序`,
	RunE: func(cmd *cobra.Command, args []string) error {
		fmt.Printf("exec %s\n", strings.Join(args, ""))

		if len(args) != 1 {
			return errors.New("参数错误")
		}

		// start process but don't wait it finished
		progCmd := exec.Command(args[0])
		progCmd.Stdin = os.Stdin
		progCmd.Stdout = os.Stdout
		progCmd.Stderr = os.Stderr
		progCmd.SysProcAttr = &syscall.SysProcAttr{
			Ptrace: true,
		}

		err := progCmd.Start()
		if err != nil {
			return err
		}

		// wait target process stopped
		pid = progCmd.Process.Pid

		var (
			status syscall.WaitStatus
			rusage syscall.Rusage
		)
		_, err = syscall.Wait4(pid, &status, syscall.WALL, &rusage)
		if err != nil {
			return err
		}
		fmt.Printf("process %d stopped:%v\n", pid, status.Stopped())

		return nil
	},
	PostRunE: func(cmd *cobra.Command, args []string) error {
		debug.NewDebugShell().Run()
		// let target process continue
		return syscall.PtraceCont(pid, 0)
	},
}

func init() {
	rootCmd.AddCommand(execCmd)
}
```

### 代码测试

下面我们针对调整后的代码进行测试：

```bash
$ cd golang-debugger/lessons/0_godbg/godbg && go install -v
$
$ godbg exec ls
exec ls
process 2479 stopped:true
godbg> exit
cmd  go.mod  go.sum  LICENSE  main.go  syms  target
```

首先，我们进入示例代码目录编译安装godbg，然后运行`godbg exec ls`，意图对PATH中可执行程序`ls`进行调试。

godbg将启动ls进程，并通过PTRACE_TRACEME让内核把ls进程停下（通过SIGTRAP），可以看到调试器输出`process 2479 stopped:true`，表示被调试进程pid是2479已经停止执行了。

并且还启动了一个调试回话，终端命令提示符应变成了`godbg> `，表示调试会话正在等待用户输入调试命令，我们这里还没有实现真正的调试命令，我们输入`exit`退出调试会话。

当我们退出调试会话时，会通过`ptrace(PTRACE_COND,...)`操作来恢复被调试进程继续执行，也就是ls正常执行列出目录下文件的命令，我们也看到了它输出了当前目录下的文件信息`cmd go.mod go.sum LICENSE main.go syms target`。

`godbg exec <prog>`命令现在一切正常了！

### 参考内容：

- Playing with ptrace, Part I, Pradeep Padala, https://www.linuxjournal.com/article/6100
- Playing with ptrace, Part II, Pradeep Padala, https://www.linuxjournal.com/article/6210
- Understanding Linux Execve System Call, Wenbo Shen, https://wenboshen.org/posts/2016-09-15-kernel-execve.html

