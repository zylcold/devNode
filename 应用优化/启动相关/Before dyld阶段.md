用户点击图标之后，会发送一个系统调用 execve 到内核，内核创建进程。 **接着会把主二进制 mmap 进来，读取 load command 中的 LC_LOAD_DYLINKER，找到 dyld 的的路径** 。然后 mmap dyld 到虚拟内存，找到 dyld 的入口函数 `_dyld_start` ，把 PC 寄存器设置成 `_dyld_start` ，接下来启动流程交给了 dyld。

注意这个过程都是在内核态完成的，这里提到了 PC 寄存器，PC 寄存器存储了下一条指令的地址，程序的执行就是不断修改和读取 PC 寄存器来完成的。