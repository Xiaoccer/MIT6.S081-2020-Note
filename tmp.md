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
4. consolewrite函数：拿cons锁？，copy字符，然后调用uartputc，然后释放锁。
5. uartputc函数在uart.c中，uart有一个循环缓冲区。uartputc会先获取uart_tx_lock锁。如果buffer满了，会调用sleep，sleep中有很多知识，后续会讲。否则，将字符放入buffer中，更新写指针，然后调用uartstart函数。最后释放uart_tx_lock锁。
6. uartstart函数：top-and bottom- half都会调用。若设备正忙，会先不发送数据，直接返回，此时数据还会在buffer中等待发送。若设备不忙，则从读指针读取数据，然后根据读指针，wakeup读指针？最后将数据写入寄存器中。
7. 但设备将数据写完后，会发生中断。在usertrap中，调用devintr函数判断是什么中断，如果是外部中断，调用plic_claim函数返回具体是哪个外部中断，若发现是UART0_IRQ中断，则调用uartintr函数。
8. 在uartintr函数中。

