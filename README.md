# Summary-about-Interrupts-Locks-and-Scheduling.
***因为越往后学，发现后续的知识和前面仍有较大的关联；而且时间久了前面的知识、细节容易遗忘，对理解后面的内容造成影响。故此将3章 Interrupts and device drivers、Locks、Scheduling 内容进行总结，以供后续查询。***

> ## 1. Interrupts and device drivers  
>> 1.驱动程序的概念：
>>> 设备驱动程序可以完成2个half的内容：  
>>> (1)top half通常被读写操作调用，让硬件开始一个行为，然后自己阻塞等待；  
>>> (2)bottom half是在硬件完成某一操作、发出中断时，进行相应的中断处理程序(interrupt handler)，中断处理程序通常是唤醒等待的进程、之后硬件继续执行其他操作。

>> 2.RISCV对于中断启用的配置过程  
>>> (1)对硬件编程，使硬件可以发送中断  
>>> (2)SIE(Supervisor Interrupt Enable，中断启用寄存器) 寄存器的配置(具体包括设置一些位：SIE_SEIE(外部中断)、SIE_SSIE(软件中断)、SIE_STIE(时钟中断)等)，在"start.c"中配置，表示内核启用相关中断。  
>>> (3)配置PLIC(Platform Level Interrupt Controller)。因为中断是PLIC负责发送的，所以需要设置PLIC表示可以处理哪些类型的中断；另一方面，"plic.c"中为每个CPU设置可以处理哪些中断，这样PLIC才可以把对应的中断发送给相应的CPU。  
>>> (4)SSTATUS(Supervisor Status) 寄存器的配置(其中的一位SSTATUS_SIE表示该CPU启用/禁用中断)，位于CPU上，因为每个CPU都可以主动启用(intr_on)、禁用(intr_off)中断。  
>>> (5)其他的相关寄存器：
>>> SIP(Supervisor Interrupt Pending)，中断判定寄存器，进程可以根据该寄存器的内容得知此次中断的类型；
>>> STVEC寄存器，进入kernel时pc的切换地址，
>>> 同理MTVEC是进入"machine mode"时pc的切换地址；
>>> SCAUSE寄存器保存了此次trap的原因  

>> 3.关于xv6中的uart(真实的物理设备)和console(虚拟设备)，因此两者都有各自的驱动程序(uart.c和console.c)，两者的大致关系如下图：
>>> ![](https://github.com/2351889401/Summary-about-Interrupts-Locks-and-Scheduling./blob/main/images/uart.png)
>>> 图中：RHR寄存器，1字节，完成"<--"方向的数据传输，当RHR寄存器收到1字节数据产生中断；THR寄存器，1字节，完成"-->"方向的数据传输，当THR寄存器传输完成时产生中断
>>>   
>>> (1) console.c  
>>> (a) cons.buf: 用于存储从RHR读取到的数据。当读到一行结束、文件末尾、"buf"缓冲区满时，唤醒"consoleread"  
>>> (b) consolewrite: 通过调用"uartputc"向"uart"的缓冲区"uart_tx_buf"写数据  
>>> (c) consoleintr: 当"uart"的RHR寄存器收到数据时，"uart"产生中断，"consoleintr"随后被调用，进行相应的字符读取  
>>> (d) consoleread: "sleep"以等待缓冲区的数据到达，并将数据拷贝到用户/内核空间
>>>  
>>> (2) uart.c  
>>> (a) uart_tx_buf: 用于存储"consolewrite"写入的数据，以便THR寄存器传输  
>>> (b) uartintr: 主要完成2个任务，当RHR收到数据时，将数据传递给"consoleintr"；当THR发送完数据时，可以继续用来发送数据  
>>> (c) uartputc: 如果"uart_tx_buf"有空间，向该"buf"中放入数据，并尝试发送数据("uartstart"被调用)；**否则会阻塞"sleep"，直到被中断处理程序唤醒(为什么? 因为阻塞意味着先拿到了互斥锁、且THR寄存器正在发送数据，此时，没有其他线程可以调用到"uartstart"，
>>> 唯一的方法是等待前一次传输完成，产生中断信号，在中断处理程序中再次调用"uartstart"释放空间，唤醒等待的线程去输入字符到缓冲区)**  
>>> (d) uartstart: 当"uart_tx_buf"非空、且THR寄存器空闲时才进行发送；之后去"wakeup"可能"sleep"的"uartputc"的内核线程
>>> (e) 将数据写入"uart_tx_buf"，与从"buf"中取出数据给THR寄存器传输，可能存在速度的差异 (因此，"uartintr"更重要的意义在于，**可能THR寄存器发送数据慢，造成"uart_tx_buf"中数据有堆积，后续的中断信号引起的中断处理程序负责将堆积的数据完整的发送出去**)  
>>> (f) 对于(e)中提到的速度差异，(e)中说的是THR传输慢；也可能的情况是THR传输很快，而将数据放入"uart_tx_buf"这一过程较慢，此时会产生很多的无谓的中断信号，如果进行相应的中断处理是没有必要的，因为**实际上不需要任何操作**，反倒是中断引起一些寄存器的切换、一些额外的指令执行很浪费时间  
>>> (g) 对于是否需要中断、使用中断还是轮询(polling)的一些总结：  
>>> (g1)高性能设备比如高性能以太网卡，如果对每个收发的数据包都产生中断，太浪费时间，因此，高性能设备尽量使用轮询  
>>> (g2)轮询会导致设备空转一段时间而不做任务，低性能设备尽量不要去浪费时间，应当使用中断  
>>> (g3)负载很高的设备，使用中断可能会加重负载，此时应当使用轮询  
>>> (g4)低负载设备，使用轮询会浪费时间，应当使用中断更好地响应事件  
>>> (g5)很多设备在中断和轮询的状态间合理切换

>> 4.timervec  
>>> 主要做的事情是：当应当发生时钟中断的时刻到达时，切换到机器模式(machine mode)下，在某个固定地址"CLINT_MTIMECMP(hart)"写入下一次时钟中断的时间，并发出时钟中断，回退到之前的状态。接着就是正常的处理时钟中断的流程(usertrap/kerneltrap
>>> --> yield --> swtch ...)

>> 5.printf (kernel中的printf)   
>>> 主要是通过获取"pr.lock"控制多CPU输出的有序性。它的主要任务是通过调用"consputc"向显示器"screen"输出内容。  

>> 6.用户空间和内核空间"printf"的区别  
>>> "uartputc"和"uartputc_sync"分别是用户空间和内核空间"printf"最终调用的函数，区别在于是否启用中断。"uartputc"启用中断，而"uartputc_sync"不启用，可能是因为内核空间的输出信息需要快速被注意到。  
>>> 用户空间的"printf": printf --> write --> sys_write --> filewrite --> consolewrite --> uartputc (cons.lock)  
>>> 内核空间的"printf": printf --> consputc --> uartputc_sync (pr.lock)  

> ## 2. Locks  
>> 1.使用锁的原因  
>>> 多个CPU上并行执行程序、单个CPU上的线程切换、中断处理程序，这些情况引起的对于共同内存的访问

>> 2.使用锁的意义  
>>> (1) locks help avoid lost updates  
>>> (2) locks make multi-step operations atmoic  
>>> (3) locks maintain an invariant ("invariant"是一些变量或属性，在执行的过程中可能会暂时违背，但是某个操作结束时能恢复到正确的情况，在单线程的情况下没有问题。但是多线程时，如果不使用lock，"invariant"会在暂时不正确的条件下，被其他线程使用，造成错误。使用"lock"，可以保护"invariant"，确保多线程程序正确运行)

>> 3.内存屏障(__sync_synchronize())  
>>> 如果没有内存屏障，编译器可能将临界区中的指令重排序到锁的外部，在并发执行时一定会发生错误；如果有内存屏障，屏障内外的指令无法越过屏障，这样才可以确保正确（需要注意的是，屏障内的指令仍可能被重排序，但这时没关系，因为处于临界区中，没有别的线程打扰，编译器即使重排序也可以保证结果的正确性）。  

>> 4.一些总结
>>> (1) 如果不是必须，不要使用锁，因为锁会降低性能；除非真的出现了"sharing"  
>>> (2) 对于锁的优化可以从"corase-grained"到"fine-grained"，但是中间过程可能会比较复杂，最后需要衡量正确性与高效性  

>> 5."spinlock"(自旋锁)与"sleeplock"(睡眠锁)
>>> (1) "spinlock"：在"acquire"和"release"之间会关中断，启用内存屏障，通过CPU自旋等待资源可用。"acquire"和"release"之间的临界区可以理解为一个原子操作，因为关中断了。
>>> (2) "sleeplock"：定义"sleeplock"的数据结构包含"spinlock"，也就是基于"spinlock"实现睡眠锁。"spinlock"必须在资源可用时才能获取到，而"sleeplock"可以在资源不可用时获取内部定义的"spinlock"；但是，由于资源实际不可用，所以会主动释放内部的"spinlock"，进入"sleep"状态，因为现在没有可用资源，应当主动放弃CPU。
>>> 这样，其他线程的"acquiresleep"也可以进行了，只是由于没有资源，都会进入"sleep"的状态。
>>> "acquire"和"release"之间为原子操作，而"acquiresleep"和"releasesleep"之间不应当是原子操作，因为会主动释放CPU。

> ## 3. Scheduling  
>> 1. "scheduler()"函数的核心"swtch(struct context* old, struct context* new)"的思想  
>>> 保存旧的内核线程的上下文、加载新的内核线程的上下文。
>>> 这里的上下文，是指线程执行过程中需要的pc、registers、栈。其中，寄存器状态易失，而"栈"的内容一般处于内存(p->kernel_stack)中，不易失。因此，对于"上下文"的保存，**核心是保存好寄存器的状态**，至于栈内容的查找，依赖于"sp"寄存器即可查找到栈的位置。
>>> 另外，上述的"swtch()"函数是汇编语言编写的函数，在内核代码中仅保存了"callee-saved registers"，没有保存"caller-saved registers"。本质上，无论是caller-saved registers还是callee-saved registers，在函数调用的过程中都是保存在栈上，只不过一类由caller函数负责保存和恢复，另一类由callee函数负责保存和恢复。如果是C语言函数调用C语言函数，编译器会帮助我们隐藏caller-saved和callee-saved寄存器的保存和恢复过程；而现在的情况是C语言函数调用汇编语言函数"swtch()"，编译器仅完成了caller-saved寄存器的保存，而callee-saved寄存器的保存和恢复需要在汇编函数"swtch()"实现。

>> 2. "sleep"和"wakeup"机制，避免"lost wakeup"  
>>> 以信号量的P和V操作为例，"sleep"和"wakeup"的主要流程是相似的:  
>>> 在P操作的"sleep"释放"s->lock"之前，获取"p->lock"；这样，即使V操作获得了"s->lock"，想要执行"wakeup"("wakeup"也需要"p->lock")，但是由于P操作之前已经先拿到了"p->lock"，所以"唤醒"操作直到P操作真正完成"睡眠"才可以进行。这样，避免了"lost wakeup"。当然，在P操作的"sleep"返回时会重新获取"s->lock"。  
```
void P(struct semaphore* s) {
  acquire(&s->lock);
  while(s->count == 0) sleep(s, &s->lock);
  s->count -= 1;
  release(&s->lock);
}

void V(struct semaphore* s) {
  acquire(&s->lock);
  s->count += 1;
  wakeup(s);
  release(&s->lock);
}
```  

>> 3. "reparent(struct proc* p)"  
>>> (1) 函数中没有首先获取"pp->lock"，因为在调用该函数之前，会获取p和p的父进程的lock("exit"函数中)。所以在扫描process table时，不应当获取每个进程的lock了，避免死锁。  
>>> (2) 在使用"pp->parent == p"之前没有使用pp->lock以确保正确性：这是因为我们只想要当前进程的子进程；并且对子进程的修改仅仅发生在我们后续的行为中。所以**对于当前进程的子进程，修改之前的读取状态一定是正确的**。
>>> (3) 发现了当前进程的子进程，通过"加锁"去修改子进程的父进程为"init"进程。而"init"进程的行为是处于死循环状态，通过"wait"系统调用回收子进程的资源。所以将进程p的子进程托付给init进程后，即使不唤醒init进程，后续init进程也能实现正确的回收子进程资源。(需要注意的一点是，"init"进程可能会存在很多的额外的子进程)

>> 4. "exit()"：进程退出时调用的函数，下面是其主要的任务
>>> (1) 回收进程的文件系统相关的资源  
>>> (2) 唤醒init进程回收子进程资源 (因为init进程永远进行wait系统调用，处于sleep的状态)。(一个进程表中实际上可能同时有很多进程处于zombie状态 (比如一个进程fork了很多子进程，但没有wait回收资源，只能使它们处于zombie状态)，它们最终的结局是被托付给init进程，所以在每个进程exit时，都唤醒init进程回收资源是很有必要的，无论它们本身是不是init进程的子进程) --> 实际上，如果一个父进程结束了早，子进程还没结束，会把子进程托付给init进程，这种情况的子进程为"孤儿进程"；如果一个父进程没有调用"wait"回收子进程资源，子进程结束时处于"zombie"状态，父进程结束时同样会将子进程托付给init进程，这种情况的子进程为"僵尸进程"。对于上述2种情况，都会导致init进程有很多子进程，因此，多次**唤醒init进程回收资源是很有必要**。  
>>> (3) 后面要做的是首先获取当前进程的父进程的锁，因为后续会通过"wakeup1"去唤醒父进程（"wakeup1"要求执行前获取进程锁），让父进程回收当前进程的资源。
但是在这之前，首先有"original_parent = p->parent"一句。因为如果后续"获取当前进程的父进程的锁"是通过"p->parent->lock"做的话。可能会出现这样的情况，"acquire"时的"p->parent"是"旧的父进程"，而"release"时的"p->parent"是"init"进程 (这种情况发生是当前进程和父进程都执行"exit"，且父进程执行了"reparent"，导致子进程两次的"p->parent"不一致)。
此时，需要注意的一点是："original_parent"可能不是真正的当前进程的父进程，但是执行到这一点意味着旧的父进程已经成为"zombie"状态了，对它执行"acquire"、"wakeup"和"release"操作没有影响。至于当前进程的"zombie"状态后续会被"init"进程重新回收。

>> 5. "wait()"：父进程回收子进程的内存资源等，如果有子进程尚未结束，则进入"sleep"等待子进程结束。  
>>> 注意：每次调用只能回收一个子进程，这就是为什么init进程要尽可能被wakeup1。  

>> 6. "kill(pid)"
>>> 主要任务是给pid编号的进程的p->killed置为1。如果pid号进程进行系统调用进入内核，直接调用"exit(-1)"结束进程；如果pid号进程因为中断、异常进入内核，完成内核中的任务后，在返回用户空间前，通过"exit(-1)"回收该进程的相关资源；如果本身在内核中执行，被kill，也是完成内核中的任务后，在返回用户空间前，通过"exit(-1)"回收该进程的相关资源。

>> 7. "pipewrite()"和"piperead()"
>>> (1) "pipewrite()": 向管道中写入若干字符，如果管道满了，则唤醒piperead相关的线程；否则正常全部输入进管道，之后再唤醒piperead进行管道读的操作  
>>> (2) "piperead()": 当管道为空时，进行睡眠；否则读取若干字符，然后唤醒pipewrite线程进行管道写的操作
