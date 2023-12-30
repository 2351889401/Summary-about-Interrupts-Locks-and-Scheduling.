# Summary-about-Interrupts-Locks-and-Scheduling.
***因为越往后学，发现后续的知识和前面仍有较大的关联；而且时间久了前面的知识、细节容易遗忘，对理解后面的内容造成影响。故此将3章 Interrupts and device drivers、Locks、Scheduling 内容进行总结，以供后续查询。***

> ## 1. Interrupts and device drivers  
>> 1.驱动程序的概念：
>>> 设备驱动程序可以完成2个half的内容：  
>>> (1)top half通常被读写操作调用，让硬件开始一个行为，然后自己阻塞等待；
>>> (2)bottom half是在硬件完成某一操作，发出中断时，进行的相应的中断处理程序(interrupt handler)，通常是唤醒等待的进程、继续执行等待硬件的操作。

>> 2.RISCV对于中断启用的配置过程  
>>> (1)对硬件编程，使硬件可以发送中断  
>>> (2)SIE(Supervisor Interrupt Enable，中断启用寄存器) 寄存器的配置(具体包括设置一些位：SIE_SEIE(外部中断)、SIE_SSIE(软件中断)、SIE_STIE(时钟中断)等)，在"start.c"中配置，表示内核启用相关中断。  
>>> (3)配置PLIC(Platform Level Interrupt Controller)。因为中断是PLIC负责发送的，所以需要设置PLIC表示可以处理哪些类型的中断；另一方面，"plic.c"中为每个CPU设置可以处理哪些中断，这样PLIC才可以把对应的中断发送给相应的CPU。  
>>> (4)SSTATUS(Supervisor Status) 寄存器的配置(其中的一位SSTATUS_SIE表示该CPU启用/禁用中断)，位于CPU上，因为每个CPU都可以主动启用(intr_on)、禁用(intr_off)中断。
>>> (5)其他的相关寄存器：SIP(Supervisor Interrupt Pending)，中断判定寄存器，进程可以根据该寄存器的内容得知此次中断的类型；STVEC寄存器，进入kernel时pc的切换地址，
>>> 同理MTVEC是进入"machine mode"时pc的切换地址；SCAUSE寄存器保存了此次trap的原因  

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

> ## 3. Scheduling  
>> 1. 
