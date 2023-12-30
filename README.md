# Summary-about-Interrupts-Locks-and-Scheduling.
***因为越往后学，发现后续的知识和前面仍有较大的关联；而且时间久了前面的知识、细节容易遗忘，对理解后面的内容造成影响。故此将3章 Interrupts and device drivers、Locks、Scheduling 内容进行总结，以供后续查询。***

> ## 1. Interrupts and device drivers  
>> 1.驱动程序的概念：
>>> 设备驱动程序可以完成2个half的内容：  
>>> (1)top half通常被读写操作调用，让硬件开始一个行为，然后自己阻塞等待；
>>> (2)bottom half是在硬件完成某一操作，发出中断时，进行的相应的中断处理程序(interrupt handler)，通常是唤醒等待的进程、继续执行等待硬件的操作。

>> 2.关于xv6中的uart(真实的物理设备)和console(虚拟设备)，因此两者都有各自的驱动程序(uart.c和console.c)：
>>> 
