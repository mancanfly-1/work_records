# HV6-RSICV系统分析

## 分析依据
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;系统代码主要是porting hv6 to RISCV小组的porting report和几个不同分支的代码仓库；对比了已有的hyperkernel代码与hv6-riscv的代码；同时，借助了石振兴同学在将ucore porting to riscv的移植报告。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;系统x86利用虚拟化(两套页表)实现隔离，内核中还要求是identity mapping；riscv中的实现用了一套页表，只不过是identity mapping，这就说明了需要假设内核中虚拟地址转化为物理地址是正确的。是否正确？

## 系统启动过程分析

1. init_syscalls() 系统调用初始化
初始化syscalls数组，将syscall函数地址存储到对应syscalls数组对应的index中。
2. arch_init() 系统体系结构初始化; 有很多驱动程序都没有移植，直接删除掉了。
    2.1 porte9_init	// 将dev设备插入到devs链表中
    - cga_init	//  将cga设备插入到devs链表中。但是前两个操作的执行函数outsb中，执行的是whie(1); 而hv6中对应执行的汇编代码
    - tsc_init	// 设置静态变量tsc_mhz=3598
    - trap_init	// 涉及到的寄存器是什么意思？细致了解riscv的寄存器spec。
        - write_csr(sscratch, 0);  //Supervisor Scratch Register; Set sscratch register to 0, indicate the exception vector that we are executing in the kernel
        - write_csr(stvec, &__alltraps); // 设置异常向量表的地址；stvec是中断向量的基地址，__alltraps在trap_entry.S中，在跳转到制定的中断处理函数之前，需要做SAVE_ALL，move a0, sp，最后jal trap。其中trap函数在trap.c中。具体的执行过程如下：<br>
        ```asm
            .globl __alltraps
        __alltraps:
            SAVE_ALL

            move  a0, sp
            jal trap
            # sp should be the same as before "jal trap"

            .globl __trapret
        __trapret:
            RESTORE_ALL
            # return from supervisor call
            sret
 
            .globl forkrets
        forkrets:
            # set stack to this new process's trapframe
            move sp, a0
            j __trapret
        ```
        - set_csr(sstatus, SSTATUS_SUM);/* Allow kernel to access user memory */
        - set_csr(sie, MIP_SSIP); // sie(中断寄存器)允许键盘中断。但是ssip是软中断的意思啊！应该是允许软中断。typora m本地编辑。
    - hvm_init      // 通过hvm来保存上下文信息，以及在切换的时候，保存寄存器中信息。
    hvm_init 所涉及到的主要结构体如下
    ```c
    typedef struct
    {
		uintptr_t user_entry;		//用户应用程序入口
		uintptr_t ulib_entry;		// ulib库的入口
	    pid_t pid;
		struct context context;		// 当前上下文
		struct trap_frame *tf;		// 当前中断帧
    } hvm_t;
    struct context {
        uintptr_t ra;
        uintptr_t sp;
        uintptr_t s0;
        uintptr_t s1;
        uintptr_t s2;
        uintptr_t s3;
        uintptr_t s4;
        uintptr_t s5;
        uintptr_t s6;
        uintptr_t s7;
        uintptr_t s8;
        uintptr_t s9;
        uintptr_t s10;
        uintptr_t s11;
    };
    struct trap_frame {
        struct push_regs gpr;			//在这里没有列出来，就是所有reg组成的机构体
        uintptr_t status;
        uintptr_t epc;
        uintptr_t badvaddr;
        uintptr_t cause;
    };
    ```
    - hvm_init的作用是设置关于hvm相关操作的一些函数指针。包含如下函数：
        - riscv_hvm_user_init函数说明<br>
设置hvm_t-->user_entry; hvm_t-->ulib_entry;
其中，user_entry = load_elf(pid, _binary_init_start); ulib_entry = load_elf(pid, _binary_ulib_start); 通过.incbin "init"和.incbin "ulib"两个汇编指令，将init和ulib文件包含到系统中来。其中init是由init.c生成的；ulib是由userlib文件夹下的所有文件生成的二进制文件。入口函数是ulib_entry.S，通过调用ulib.c文件中的ulib_init接口来完成。
        - riscv_hvm_run_user_init函数说明<br>
在运行用户进程之前，需要设置页表根目录到satp(x86中的cr3)中；在刷新tlb之后，就可以执行ulib_entry，完成用户库的加载工作。
        - riscv-hvm_switch<br>
切换hvm的时候一定先将satp设置到目标进程(hvm)的page table root；
kernel/switch.S中定义了switch_to的方法, 就是将src_hvm中的寄存器中的内容保存到栈中，将dest_hvm中栈中的contextload进寄存器中。
        - riscv_hvm_flush<br>
do nothing
        - riscv_hvm_copy<br>
riscv_hvm_copy(void *_dst, void *_src, pid_t pid)
将hvm_src拷贝到hvm_dst中，但是，需要重置dst->pid = pid; 
dst->tf->gpr.a0和sp
dst->context.sp和ra	//内核栈的切换，和中断时栈的变化。
        - riscv_hvm_set_pid<br>
设置hvm->pid
        - riscv_hvm_invalidate_tlb<br>
asm volatile ("sfence.vma");		//调用sfence.vma这个命令是用于刷cache的。
    - iommu_init<br>
        iommu 啥也没做，riscv不支持iommu
    - iommu_early_set_dev_region<br>
        do nothing
    - reserve_and_allow_ports<br>
        m1、com2、keybord、CRT			//可能不存在。
    - reserve_vectors<br>
        初始化设备vector_table。
- vm_init() 虚拟地址空间初始化<br>
根据page_desc_table生成链表；并且，设置sealboot为true。
- user_init() 
    - 创建初始进程，pid= INITPID
    - 调用setup_kernel_map方法，将整个内核空间进行映射，映射的方式是identity map。这里使用了新创建的系统调用 sys_map_page, 这里是如何进行恒等映射的还需要仔细分析。setup_kernel_map函数调用了page_walk函数， 该函数的目的是为kernel创建四级页表；
    - load用户init程序和ulib，并分配页；user_entry = load_elf(pid, _binary_init_start);  ulib_entry = load_elf(pid, _binary_ulib_start);
    - 映射page_desc_table
    - 执行arch_user_init()方法，映射内核的command line 	//map kernel command line
    - 调用hvm_user_init，设置user_entry和ulib_entry到对应的处理函数
    - 将hvm设置为当前pid，设置pid的状态为running，将当前pid添加到运行状态的链表中。
- write_csr()
    - 设置为调式模式
- hvm_run_user_init()			//运行当前进程
    - 获得当前进程的page_table_root,写入到satp寄存器中。开启页表
    - 刷新tlb// 
    - 根据hvm中用户程序的地址，执行用户程序。

----
## 关键信息纪要

1. 关于页表
   | 位       | 名称   | 意义                                                    			|<br>
   | ---------| -------| --------------------------------------------- 				        |<br>
   | 0        | V      | 1代表PTE有效，0代表PTE无效                              			|<br>
   | 1        | R      | 可读。                                                  			|<br>
   | 2        | W      | 可写。                                                 			|<br>
   | 3        | X      | 可执行。                                                			|<br>
   | 4        | U      | 1代表U-mode可访问，0代表U-mode不可访问（仅用于叶子PTE）            |<br>
   | 5        | G      | 全局映射                                                			|<br>
   | 6        | A      | 虚拟页曾被访问过（仅用于叶子PTE）                       			|<br>
   | 7        | D      | 虚拟页曾被写入过（仅用于叶子PTE）                      			|<br>
   | 8 ～ 9   | RSW    | reserved                                                			|<br>
   | 10 ～ 18 | PPN[0] | 四级页目录的物理页编号                                  			|<br>
   | 19 ～ 27 | PPN[1] | 三级页目录的物理页编号                                  			|<br>
   | 28 ～ 36 | PPN[2] | 二级页目录的物理页编号                                  			|<br>
   | 37 ～ 53 | PPN[3] | 一级页目录的物理页编号                                  			|<br>
   | 54 ～ 63 |        | reserved                                  			                |<br>
1. 首先，页表标识位与x86是不同的；同时，在非叶子页表的情况下，page table entry的x/w/r位都为0。RV64可以使用的内存管理方案有Sv39和Sv48两种。Riscv-HV6将使用Sv39内存管理方案。Sv39是基于页的39位虚拟内存系统。Sv39支持39位的虚拟地址空间，其设计遵循Sv32的总体方案，由Sv32的二级映射换到了三级映射。
2. Sv39的虚拟地址是64位的，但只有低39位有效，且高25位必须和第38位相等（注：这里说的第38位指的是从0开始计数）。从虚拟地址到物理地址共需3级页表的转换。每一级的PTE都有可能是叶子PTE，所以除了4K的页外，还有可能存在2M或1G的页，每种页都必须依照它自己的大小在虚拟层面和物理层面对齐。
3. Sv48是目前hv-riscv使用的虚拟内存管理方式，它使用了四级页表，与hv6-x86一致。
对于console的输入输出，是通过何种方式来设置的。中断？分析时，发现是通过ecall命令访问执行环境。sbi.h中，#define SBI_CONSOLE_PUTCHAR 1 将其值放入到a7寄存器中，是意味着这是个中断号码？
sscratch：Set sscratch register to 0, indicating to exception vector that we are presently executing in the kernel
----
## 进程
- 没有调度系统?；XV6的操作系统架构分析。
- fork    
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;fork过程详细分析，在这里过程调用了很多syscall，包含内存和hvm相关内容，可以很好的理解系统调用.
1. 申请3个page分别对应pml4、stack和hvm；以及一个未使用的pid
2. 调用sys_clone, 复制current进程中的pml4、stack和hvm到上面申请的页中；sys_clone是通过hv6/entry.S调用的。
3. 在赋值hvm之前要刷新current 和 children的hvm。但是riscv_hvm_flush do nothing，就是不用刷新。拷贝hvm调用的方法是riscv_hvm_copy，包括制父进程的上下文，并且设定返回值 a0 和context的返回地址 ra等。
4. 拷贝文件描述符，这里需要调用sys_dup系统调用
5. 拷贝page_table。这个过程需要调用系统调用sys_alloc_pdpt、sys_alloc_pd、sys_alloc_pt和sys_alloc_frame。即完整的拷贝四级页表
6. 设置申请的进程为runnable状态后，返回。<br>
由于fork是在用户空间调用的，所以，调用的系统调用是通过ulib库完成的，ulib库中的对应系统调用函数要访问usys.S中的对应调用，这些调用实现的方式是通过ecall命令，trap到supervisor。
