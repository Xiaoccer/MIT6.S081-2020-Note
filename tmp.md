[toc]

# MEM

* kvminit函数(vm.c)：会将trampoline map到内核的高虚拟地址
* procinit函数(proc.c)：会为每个进程proc分配一页kstack，然后map到内核的虚拟地址上
* allocproc函数(proc.c)：会分配一页trapframe，让proc指向它。然后会创建一个页表，将trampoline map到user的高虚拟地址上；将trapframe也map到第二高地址上

#  Trap

寄存器为什么不能直接保存在用户栈上，因为不同的编程语言可能会用不同的栈分布，所以为了兼顾不同的编程语言。

## 用户态通过系统调用进入trap

1. 首先调用``usys.pl``会生成一个汇编文件``usys.S``，该文件会把相应的系统调用号放入a7寄存器，然后调用``ecall``
2. ecall指令会设置成管理者模式，然后将当前程序计数器pc保存在sepc中，然后让当前pc = stvec
3. stvec保存的是trap的地址，有内核trap和用户态trap，这里保存的是用户态trampline.S的地址，什么时候保存的？
4. 此时CPU跳转到trampoline.S的代码中，但是还没有使用内核页表和内核栈。此时ssratch保存了进程的trapframe地址，什么时候保存的？猜测是切换进程时候保存的。接着执行trampoline.S中的uservec代码。
5. 在uservec代码中，先把a0寄存器和sscratch交换，此时a0寄存器就执行trapframe，接着用trapframe的第40个字节开始保存当前寄存器的值。然后将t0和sscratch寄存器交换，此时t0保存的是a0原本的值，将t0保存到相应位置。接着开始从trapframe中加在sp(kernel_sp)，tp(kernel_hartid),t0(kernel_trap)和t1(kernel_satp)。这些是什么时候初始化的？接着交换t1和satp，切换置内核页表。直接jump t0，此时t0保存的是kernel_tap（也是usertrap）。
6. usertrap函数（trap.c）：首先判断中断是否来自user，将stvec地址设置为kernelvec.S（调用kerneltrap）。由于现在在内核状态，所以先调用myproc获取当前进程信息，然后将sepc的内容（此时存的是当前pc）保存在该进程的trapframe->epc中。判断如果是系统中断，则epc+4，用作返回下一条指令。然后打开设备中断（每次在调度时，acquire锁会先关闭中断） 然后调用syscall。
7. syscall会取a7这个参数，得到具体需要调用哪个系统调用，然后进行调用，最后将返回值存入a0寄存器。
8. 最后调用usertrapret()函数：首先关掉中断（为了正确设置trapframe中kernel相关的寄存器），然后设置stvec地址为uservec，设置trapframe中的kernel_satp，kernel_sp，kernel_trap和kernel_hartid。设置sstatus（用户可中断和sret返回的是用户模式），设置sepc为trapframe->epc。调用trampoline.S中的userret，并且传入trapframe地址和用户的页表作为参数。
9. 在userret中，a0保存trapframe，a1保存用户页表。所以先拷贝a1置satp，切换成用户页表。接着将旧的a0值存入sscratch（作用是啥？）。因为现在a0保存的是trapframe，所以能直接从a0的各个地址中恢复寄存器的内容。最换交换sscratch和a0的值，所以结果是a0恢复成原来的a0值，sscratch存的是trapframe的值。最后调用sret返回用户pc.
10. the RISC-V always disables interrupts when it starts to take a trap, and xv6 doesn’t enable them again until after it sets stvec.

## 内核态进入trap

1. 此时stevc指向的是kernelvec.S，该代码会保存一系列寄存器。然后调用kerneltrap函数
2. kerneltrap函数会判断是中断属于设备中断还是内核中断，若是timer的中断，还有调用yield函数，进行线程的切换，具体细节后面章节会分析。
3. 当线程切换回来后，要重新加载sepc和sstatus，然后恢复寄存器。

# 外部中断(UART为例)

1. 机器启动的时候，会调用start函数(start.c)，然后设置启用外部中断，内部中断和定时器中断。SEIE、STIE和SSIE。
2. 然后跳入main.c对每个cpu进行特定的寄存器的编程。注意的是，所有的cpu都会执行main.c的程序。有些硬件的寄存器只需要编程一次，所以只在cpu_id为0的执行一次性的操作，比如初始化打印设备consoleinit()。
3. consoleinit函数调用uartinit函数，配置的uart芯片。此时uart芯片可以产生中断，但还没有对PLIC编程，所以还没有开启中断服务，因此需要调用plicinit函数，consoleinit函数还在devsw数组中保存了consoleread函数和consolewrite函数，devsw数组是devsw类型的，用于保存设备read和write函数。plicinit函数直接针对硬件地址进行编程设置。注意**consoleinit和plicinit只需要执行一次**，所以只在cpu0中执行。
4. 接着调用plicinithart函数，该函数是**每个cpu都要调用**，用来设置每个cpu都可以服务uart中断。
5. 接着每个cpu在scheduler函数中，设置sstatus寄存器，开启接受中断服务。

## console如何显示美元符号的

1. 首先在第一个程序init.c中创建了一个文件为console，设备类型为T_DEVICE，用于代表终端，0，1，2三个描述符都指向该文件。然后执行sh.c。
2. sh.c中的getcmd函数会调用fprintf函数向描述符2写入，为什么是2？fprintf又会调用vprintf，最终调用write函数。
3. write函数调用sys_write，然后调用filewrite函数。在filewrite中，会判断文件描述符的类型，如果是FD_DEVICE类型，就会从devsw数组中调用consolewrite函数。
4. consolewrite函数：拿cons锁，copy字符，调用uartputc，然后释放cons锁。cons是一个结构体，里面含有一个锁和一个buf。conf可以理解为uart的接收数据循环缓冲区（键盘是生产者，shell是消费者）。
5. uartputc函数在uart.c中，uart有一个循环缓冲区，可以理解为发送缓冲区（shell是生产者，console是消费者）。uartputc会先获取uart_tx_lock锁。如果buffer满了，会调用sleep，sleep中有很多知识，后续会讲。否则，将字符放入buffer中，更新写指针，然后调用uartstart函数。最后释放uart_tx_lock锁。
6. uartstart函数：top- and bottom- half都会调用。若设备正忙，会先不发送数据，直接返回，此时数据还会在buffer中等待发送。若设备不忙，则从读指针读取数据，然后根据读指针，wakeup读指针？最后将数据写入寄存器中。
7. 当设备将数据写完后，会发生中断。在usertrap中，调用devintr函数判断是什么中断，如果是外部中断，调用plic_claim函数返回具体是哪个外部中断，若发现是UART0_IRQ中断，则调用uartintr函数。
8. 在uartintr函数中，如果有字符可读，会先调用uartgetc函数获取字符，然后调用consoleintr函数（显然初始化显示美元符号的时候，是没有字符可读的）。接着获取uart_tx_lock锁，调用uartstart函数发送buffer中的字符，然后释放uart_tx_lock锁。注意，当console一次性写入多个字符时，只有第一个字符是通过uartputc函数中的uartstart发送，其余的在第一个字符发送完毕后的中断中调用uartintr的uartstart发送。
9. consoleintr是对读取字符进行处理，下面会详细讲到。

## 从键盘中读取字符

1. sh.c中的gets会调用read函数，read调用sys_read调用fileread，进而调用consoleread。
2. consoleread函数中，首先获取cons锁，若没有数据就sleep；否则从cons.buf中读取数据，拷贝到相应地方，最后释放cons锁。
3. 如上所说，只要产生了uart的中断，都会在uartintr中调用uartgetc函数看是否有字符可读，然后就调用consoleintr函数。
4. consoleintr函数会获取cons锁，然后对从键盘读入的字符进行处理，对于特殊的字符如ctrl-p之类的，就执行相应操作；其余的会先调用consputc将字符回显到console上，然后更新cons.buf供shell读取。
5. consputc采用的是同步put，会关闭中断，然后向uart寄存器写入字符。
6. cons的buf有三个指针，r,w和e，为什么有e，盲猜和退格换行等有关，有空可钻研一下。
7. 如果用户没有输入完整的一行，任何读键盘的进程会一直sleep。因为consoleintr中只有整行数据到了，才会wakeup sleep中的进程。

## 计时器中断

1. start.c函数中会有timerinit的调用，该函数对CLINT进行了编程，还将机器模式下的trap handler置为timervec函数（设置mtvec为timervec）。
2. timervec会对CLINT重新编程，然后发起一个软中断，在最后mret的时候从机器模式返回管理者模式。该软中断可以被内核屏蔽的，
3. 疑问：为什么每个cpu会32个暂存区（scratch），不过只用了其中的6个。

# Lock

## spinlock

1. 是否能获得锁是靠test_and_set这个原子指令，关闭中断是为了防止同一CPU的死锁。
2. 在acquire锁的时候，会先关闭中断。原因是：
   1. 防止同一CPU造成死锁。假设该CPU在进程A获取了L锁，然后发生了定时器中断，在进程B中也想获取L锁。由于进程A的L锁未释放，进程B会一直空转cpu进行等待，该中断得不到返回，就会造成死锁。

# Scheduling

## swtch/yield

1. 在swtch之前，除了p->lock外，不应该获取其他的lock，不然会造成不同CPU之间的死锁。如CPUA获取了锁L后，进行swtch让出了进程。此时CPUB接管了该进程，然后尝试获取锁L，此时CPUB会一直空转，无法重新让出该进程，造成死锁。

## sleep/wake up (Coordination)

1. 背景：线程可能需等待一个事件，如写磁盘，写管道等。可以不断轮询，但会浪费CPU的时间（CPU切换上下文）。所以希望有其他机制可以让进程先放弃CPU，然后等事件完成后，再唤醒。
2. Sleep:传参是一个chan（一个地址）和一个锁。防止chan的值重复，一般这个地址都是全局数据，在内存只有一份。
   1. 传入锁的注意点：
      1. 若调用切换线程前，没有释放该条件锁，会造成不同CPU的死锁。
      2. 要在获取到**进程锁**之后，调用sched之前，释放该条件锁，不然会造成lost wakeup的现象。所以最终切换进程前，只获取了一个p->lock。
      3. 在sleep返回后，会重新获取该条件锁。
   2. 由于xv6中的wakeup会唤醒所有睡眠的进程，所以进程唤醒后，还要判断一些之前sleep的条件是否还成立，若成立，继续sleep。所以一般有while循环来判断一个sleep条件。

## exit/wait/kill

1. exit（主动退出）:
   1. 若父进程要退出，会让init进程接管其子进程（子进程就成了孤儿进程？）。
   2. 会wakeup其父进程，因为其父进程可能在wait。
   3. 将自己状态设置为ZOMBIE（成为僵尸进程），然后跳到sched。
2. wait:
   1. 扫描属于本父进程的子进程，如果有ZOMBIE状态的，再释放其资源。
3. kill（被动退出）：
   1. 被kill的过程中，可能正在执行一些原子操作，如更新文件数据结构等。为保证操作的原子性，不会立马结束进程，而是设置p->killed=1。
   2. 第一种情况进程正在运行：当该进程被定时器中断或者通过系统调入进入内核，usertrap看到p->killed=1，会调用exit函数来结束进程。
   3. 第二种情况进程正在sleep:如果被kill的进程处于sleeping状态，kill还会将进程置为runnable状态。当进程再次运行时，会恢复置sleep之后的语句，在sleep条件的循环判断中，可以判断p->killed的状态，然后做进一步的处理。若是在系统调用中，直接返回调用，这样能返回置usertrap，可结束进程。但像一些原子性操作，如等待磁盘更新的sleep，可能就不判断p->killed，等其操作完成再返回至usertrap。

# File system

## disk layer

1. 磁盘：一般能写的最小单位是扇区(sector)，一般为512字节。block大小由文件系统来决定，一般为多个扇区组成。
2. disk inode（dinode）
   1. 可以看作是文件的索引，里面有type字段指示是文件还是目录，还有nlink字段指示有多少目录指向该inode。
   2. dinode里面有data block的地址，block地址分为直接地址和间接地址。间接地址可以增大文件数据的大小。
   3. 若该inode是目录类型，它在block的data就是一些目录项。目录项一般就包含文件名和对应的inode号（所以说inode相当于文件的索引）。
   4. 若新建一个文件，首先找到空闲的inode块，然后更新根目录的inode信息（如size信息），然后更新根目录的block信息（如添加新增文件的目录项），接着更新bitmap，找到空闲的block，最后往block写入数据，并更新该文件的inode信息（如数据的block编号等）。

## Buffer cache

1. buffer cache的作用是缓存从磁盘读上去来的block数据，xv6的设计有以下特点：
   1. 通过双向链表实现lru的淘汰算法。
   2. 有两种锁，一种是cache的大锁，遍历的时候用；一种是针对每个block的锁。在获取block级别的锁之前，会先释放cache大锁。
   3. 每个block的锁获取是通过自旋锁+sleep来实现的，叫sleeplock，实现有点巧妙，建议看代码理解。简单来说，就是sleeplock内部有个自旋锁和locked标志位。要持有锁时，先获取内部自旋锁，要是获取自旋锁成功且locked为0，则成功持有锁，将locked置为1。若获取自旋锁成功但locked为1，则调用sleep睡眠，sleep会释放自旋锁的。这样的好处是，磁盘的操作一般比较耗时，无法获取锁的时候可以用sleep让出cpu，避免浪费。
