<h1><center>lab5实验报告</center></h1>

## 练习零

+ 在`alloc_proc`中添加额外的初始化：

  ```c
  proc->wait_state = 0;
  proc->cptr = NULL;
  proc->optr = NULL;
  proc->yptr = NULL;
  ```

+ 在`do_fork`中修改代码如下：

  ```c
  if((proc = alloc_proc()) == NULL) {
      goto fork_out;
  }
  proc->parent = current; // 添加
  assert(current->wait_state == 0);
  if(setup_kstack(proc) != 0) {
      goto bad_fork_cleanup_proc;
  }
  ;
  if(copy_mm(clone_flags, proc) != 0) {
      goto bad_fork_cleanup_kstack;
  }
  copy_thread(proc, stack, tf);
  bool intr_flag;
  local_intr_save(intr_flag);
  {
      int pid = get_pid();
      proc->pid = pid;
      hash_proc(proc);
      set_links(proc);
  }
  local_intr_restore(intr_flag);
  wakeup_proc(proc);
  ret = proc->pid;
  ```

## 练习一
EXERCISE1 的目标是设置陷阱帧（trapframe）以准备用户进程的执行环境。具体来说，它要做以下三个操作：

1. 设置栈指针（tf->gpr.sp）：栈指针应该指向用户栈的顶部，即 USTACKTOP。这样，在用户进程执行时，它可以正确地使用栈来保存局部变量和函数调用。
2. 设置程序计数器（tf->epc）：程序计数器应该指向用户程序的入口点，即可执行文件的 ELF 文件头（elf->e_entry）的值。这样，在用户进程执行时，它会从用户程序的入口点开始执行。
3. 设置状态寄存器（tf->status）：状态寄存器应该与之前保存的 sstatus 值相同，但需要清除 SPP 和 SPIE 位，以便将用户进程的状态设置为用户模式，并允许用户进程处理异常。
这样设置陷阱帧后，当内核将控制权转移到用户进程时，用户进程将从正确的入口点开始执行，并在用户模式下执行。

### 代码

将`sp`设置为栈顶，`epc`设置为文件的入口地址，`sstatus`的`SPP`位清零，代表异常来自用户态，之后需要返回用户态；`SPIE`位清零，表示不启用中断。

```c
tf->gpr.sp = USTACKTOP;
tf->epc = elf->e_entry;
tf->status = sstatus & ~(SSTATUS_SPP | SSTATUS_SPIE);
```

### 执行过程

1. 在`init_main`中通过`kernel_thread`调用`do_fork`创建并唤醒进程，使其执行函数`user_main`，这时该进程状态已经为`PROC_RUNNABLE`，表明该进程开始运行
2. 在`user_main`中通过宏`KERNEL_EXECVE`，调用`kernel_execve`
3. 在`kernel_execve`中执行`ebreak`，发生断点异常，转到`__alltraps`，转到`trap`，再到`trap_dispatch`，然后到`exception_handler`，最后到`CAUSE_BREAKPOINT`处
4. 在`CAUSE_BREAKPOINT`处调用`syscall`
5. 在`syscall`中根据参数，确定执行`sys_exec`，调用`do_execve`
6. 在`do_execve`中调用`load_icode`，加载文件
7. 加载完毕后一路返回，直到`__alltraps`的末尾，接着执行`__trapret`后的内容，到`sret`，表示退出S态，回到用户态执行，这时开始执行用户的应用程序

## 练习二

### 代码

首先获取源地址和目的地址对应的内核虚拟地址，然后拷贝内存，最后将拷贝完成的页插入到页表中。

```c
uintptr_t* src = page2kva(page);
uintptr_t* dst = page2kva(npage);
memcpy(dst, src, PGSIZE);
ret = page_insert(to, npage, start, perm);
```

### COW设计

+ 在`fork`时，将父进程的所有页表项设置为只读，在新进程的结构中只复制栈和虚拟内存的页表，不为其分配新的页
+ 切换到子进程执行时，如果子进程需要修改一页的内容，会访问页表，由于该页不允许被修改，所以会引发异常
+ 异常处理部分，遇到该类异常，重新分配一块空间，将访问的页面复制进去，更新子进程的页表项

## 练习三

### 函数分析

1. `fork`：通过发起系统调用执行`do_fork`函数。用于创建并唤醒进程，可以通过`sys_fork`或者`kernel_thread`调用。
   + 初始化一个新进程
   + 为新进程分配内核栈空间
   + 为新进程分配新的虚拟内存或与其他进程共享虚拟内存
   + 获取原进程的上下文与中断帧，设置当前进程的上下文与中断帧
   + 将新进程插入哈希表和链表中
   + 唤醒新进程
   + 返回进程`id`
2. `exec`：通过发起系统调用执行`do_execve`函数。用于创建用户空间，加载用户程序，可以通过`sys_exec`调用。
   + 回收当前进程的虚拟内存空间
   + 为当前进程分配新的虚拟内存空间并加载应用程序
3. `wait`：通过发起系统调用执行`do_wait`函数。用于等待进程完成，可以通过`sys_wait`或者`init_main`调用。
   + 查找状态为`PROC_ZOMBIE`的子进程；如果查询到拥有子进程的进程，则设置进程状态并切换进程；如果进程已退出，则调用`do_exit`
   + 将进程从哈希表和链表中删除
   + 释放进程资源
4. `exit`：通过发起系统调用执行`do_exit`函数。用于退出进程，可以通过`sys_exit`、`trap`、`do_execve`、`do_wait`调用。具体执行内容：
   + 如果当前进程的虚拟内存没有用于其他进程，则销毁该虚拟内存
   + 将当前进程状态设为`PROC_ZOMBIE`，唤醒该进程的父进程
   + 调用`schedule`切换到其他进程

### 执行流程

+ 系统调用部分在内核态进行，用户程序的执行在用户态进行
+ 内核态通过系统调用结束后的`sret`指令切换到用户态，用户态通过发起系统调用产生`ebreak`异常切换到内核态
+ 内核态执行的结果通过`kernel_execve_ret`将中断帧添加到进程的内核栈中，从而将结果返回给用户

### 生命周期图

```
                    +-------------+
               +--> |	 none 	  |
               |    +-------------+       ---+
               |          | alloc_proc	     |
               |          V				     |
               |    +-------------+			 |
               |    | PROC_UNINIT |			 |---> do_fork
               |    +-------------+			 |
      do_wait  |         | wakeup_proc		 |
               |         V				  ---+
               |    +-------------+       do_wait 	  	 +-------------+
               |    |PROC_RUNNABLE|    <------------>    |PROC_SLEEPING|
               |    +-------------+       wake_up        +-------------+
               |         | do_exit
               |         V
               |    +-------------+
               +--- | PROC_ZOMBIE |
                    +-------------+
```
