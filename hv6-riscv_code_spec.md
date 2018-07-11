# 代码对应规约
## 状态空间对应数据结构
- 系统状态机
``` python
class KernelState(BaseStruct):
    pages_ptr_to_int = Map(uint64_t)                    # 页指针
    proc_table_ptr_to_int = Map(uint64_t)               # 进程表指针
    page_desc_table_ptr_to_int = Map(uint64_t)          # 页描述符表指针
    file_table_ptr_to_int = Map(uint64_t)               # 文件表指针
    devices_ptr_to_int = Map(uint64_t)                  # 设备表指针
    dmapages_ptr_to_int = Map(uint64_t)                 # dma_page表指针
    # 对应到系统代码中各个系统对象数据结构
    procs = Proc()
    pages = Page()
    dmapages = DMAPage()
    files = File()
    pci = PCI()
    pcipages = PCIPage()
    vectors = Vectors()
    io = IO()
    intremaps = Intremap()
    
    current = Map(pid_t)                                # 当前运行的进程
    iotlbinv = Map(bool_t)                              # 是否刷新iotlb

    def flush_iotlb(self):
        self.iotlbinv = z3.BoolVal(True)

    def flush_tlb(self, pid):
        self.procs[pid].tlbinv = z3.BoolVal(True)       # 每个进程的tlb是否刷新
```
状态机中各种指针对应的位置
由于内核被加载到0x80000000，就是物理地址，proc_table, page_desc_table， file_table, device_table 对应的都是物理地址。

kernel start address: 80000000<br>
kernel end address: 86400000<br>

dmapages:80c00000 80e00000<br>
devices: 80e00000 80e01c00<br>
file_table:80e02000 80e03400<br>
proc_table:80e05000, 80e0ac00<br>
page_desc_table: 80e0b000 80e4b000<br>
pages: 0x80e4b000<br>

## syscall分析

### 进程管理代码分析
#### 进程数据结构和规约对比
![image](/home/zhangqiang/Pictures/Code_Proc.png)
![image](/home/zhangqiang/Pictures/Spec_Proc.png)<br>
两者重要的区别是hvm虽然是一个页，但在code中有详细的数据结构，这需要我们自己来维护。而x86中是由虚拟化技术来实现的。
#### sys_clone(pid_t pid, pn_t pml4, pn_t stack, pn_t hvm)(clone_proc):this is called by sys_clone in entry.S.
1. 根据给出的pml4, stack和hvm pages，调用alloc_proc方法来创建进程
2. 如果创建失败，退出；否则，获得current进程，将当前进程的stack拷贝到pid的stack中
3. 将current的hvm拷贝到pid的hvm里
4. 修改hvm中的关键信息,调用方法为hvm.c 中riscv_hvm_copy方法<br>
``` c
    hvm_t* dst = (hvm_t*)_dst;
	hvm_t* src = (hvm_t*)_src;
	*dst = *src;
	dst->pid = pid;

	dst->tf = (struct trap_frame *)(get_page(get_proc(dst->pid)->stack) + PAGE_SIZE) - 1;
	*(dst->tf) = *src->tf;
	dst->tf->gpr.a0 = 1;

	dst->tf->gpr.sp = src->tf->gpr.sp;
	dst->context.sp = dst->tf;
	dst->context.ra = forkret;
	/* 如上所示，如果验证通过，则说明parent_hvm.tf-->gpr.a0， parent_hvm->tf = parent_hvm->context.sp， parent_hvm->context.ra = */
	/* forkret；因为规约中给出的spec是hvm整个page都相等。所以感觉这样无法验证通过。*/
	/* 调查发现，能够验证通过的原因是对于hvm的操作，在python中都pass掉了，根本就不会影响状态空间。。。。*/
```

---

#### sys_set_proc_name<br>
  没有进行任何状态改变，只是改了名字
  
---
####  sys_reap
1. 根据pid获得该进程
2. 判断该进程的状态，是否能够释放
3. 释放,下面代码为释放的过程.对比spec，操作一致
```c
    struct proc *proc;
    proc = get_proc(pid);
    /* init cannot be freed, so ppid should be non-zero */
    --get_proc(proc->ppid)->nr_children;
    proc->state = PROC_UNUSED;
    proc->ppid = 0;
    proc->page_table_root = 0;
    proc->stack = 0;
    proc->hvm = 0;
    proc->launched = 0;
    proc->killed = 0;
    proc->use_io_bitmap = 0;
    proc->io_bitmap_a = 0;
    proc->io_bitmap_b = 0;
    proc->name[0] = 0;
    proc->name[1] = 0;
```
####  sys_switch 调用sys_switch_helper,最终调用的函数为switch_proc
1. 判断当前pid进程是否合法，如果不合法退出
2. 判断pid是否为current，如果是，直接返回true；否则进入3
3. 判断current的状态是否为running状态，是，继续4；否则进入5
4. 判断current是否被killed，是，设置当前current状态为PROC_ZOMBIE，将其从就绪队列中删除，否则设置current状态为PROC_RUNNABLE;
5. 设置pid的状态为RPOC_RUNNING,并设置current= pid，
6. 调用hvm_switch方法，切换hvm
```c
    hvm_t* from_hvm = _from_hvm;
	hvm_t* to_hvm = _to_hvm;
	lcr3(get_page(get_proc(to_hvm->pid)->page_table_root));
	switch_to(&from_hvm->context, &to_hvm->context);    //switch_to方法将原有contex信息保存到栈中，然后加载pidcontext到各寄存器。
text
.globl switch_to
switch_to:
    # save from's registers
    STORE ra, 0*REGBYTES(a0)
    STORE sp, 1*REGBYTES(a0)
    STORE s0, 2*REGBYTES(a0)
    STORE s1, 3*REGBYTES(a0)
    STORE s2, 4*REGBYTES(a0)
    STORE s3, 5*REGBYTES(a0)
    STORE s4, 6*REGBYTES(a0)
    STORE s5, 7*REGBYTES(a0)
    STORE s6, 8*REGBYTES(a0)
    STORE s7, 9*REGBYTES(a0)
    STORE s8, 10*REGBYTES(a0)
    STORE s9, 11*REGBYTES(a0)
    STORE s10, 12*REGBYTES(a0)
    STORE s11, 13*REGBYTES(a0)

    # restore to's registers
    LOAD ra, 0*REGBYTES(a1)
    LOAD sp, 1*REGBYTES(a1)
    LOAD s0, 2*REGBYTES(a1)
    LOAD s1, 3*REGBYTES(a1)
    LOAD s2, 4*REGBYTES(a1)
    LOAD s3, 5*REGBYTES(a1)
    LOAD s4, 6*REGBYTES(a1)
    LOAD s5, 7*REGBYTES(a1)
    LOAD s6, 8*REGBYTES(a1)
    LOAD s7, 9*REGBYTES(a1)
    LOAD s8, 10*REGBYTES(a1)
    LOAD s9, 11*REGBYTES(a1)
    LOAD s10, 12*REGBYTES(a1)
    LOAD s11, 13*REGBYTES(a1)
    ret
```<br>
``` python
def sys_switch(old, pid):
    cond = z3.And(
        is_pid_valid(pid),
        old.procs[pid].state == dt.proc_state.PROC_RUNNABLE,
        old.current != pid,
    )
    new = old.copy()
    new.procs[old.current].state = util.If(
        old.procs[old.current].killed, dt.proc_state.PROC_ZOMBIE, dt.proc_state.PROC_RUNNABLE)
    new.procs[pid].state = dt.proc_state.PROC_RUNNING
    new.current = pid
    return cond, util.If(cond, new, old)
```
- [x] 规约中并没有体现对context的更改。所以，这个一定不能验证通过。因为根本就没对hvm进行建模，只是一个单纯的page。
- 调查结果显示，对hvm_switch操作在python中直接pass掉了，不会改变状态空间。所以验证通过了。

---
####  sys_kill
1. 首先判断要kill的pid的合法性，包含进程pid的状态(只有embryo/runnable/running三种状态的进程能够被kill，unused or zombie不行)和pid本身值是否合法
2. 根据pid获得该进程，并设置killed属性为true
3. 如果该进程不是处于PROC_RUNNING状态，则设置该进程的状态为PROC_ZOMBIE，并将该进程从就绪队列中删除。<br>
可以看出，sys_kill并没有释放pid对应进程的数据结构，只是对killed属性进行了设置，同时进程当前判断状态，来设置kill后的状态。规约中和实现的表述是一致的。

- sys_reparent（reparent操作的含义是将当前pid进程的父亲进程指定为INITPID，pid为要reparent的进程id）
1. 判断pid是否合法，如果合法，获取pid对应的进程；否则退出
2. 判断pid对应的进程状态是否合法(非PROC_ZOMBIE),同时，INITPID进程的状态也不能为PROC_RUNNABLE和PROC_RUNNING两种状态
3. INITPID的nr_children++;pid对应的父进程的nr_children--
4. 设置pid进程的ppid为INITPID<br>
可以看出，上面的操作，只涉及到了两个进程的nr_children和ppid属性，在规约中很好对应，没有问题。

---
### 学习过程中查询了相关信息
- 程序在编译过程中的链接地址是物理地址还是虚拟地址？还是都不是？
    - 运行地址《--》链接地址
    - 存储地址《--》加载地址
    - 程序的链接地址<br>
    显然是虚拟地址。在ulib.lds中使用的是0xffffffff80000000
- 编译过程<br>
    1. 首先根据hv6/user/lib/ulib.lds，链接成ulib;
    2. 然后生成init用户程序elf;(没有指定链接脚本，所以，从0x0作为链接地址(即，运行地址))
    3. 根据kernel.lds，将hv6的内核各个目标文件链接成hv6.elf.链接地址是0x80000000。为什么使用这个地址呢？<br>
    张蔚的回答：目前该地址对应的就是物理地址。如果不使用这个地址，就需要bbl为我们提供地址的映射方案，但目前由于bbl当前版本已经不提供了，所以直接使用了这个地址，即物理地址。
    4. 使用objcopy方法将hv6.elf转换成hv6.bbl.dbg --adjust-vma 0x80000000（修改了链接地址）;
    5. cp bbl ../../hv6/o.riscv64/hv6.bbl（首先复制bbl为hv6.bbl）;
    6. riscv64-unknown-elf-objcopy -O binary o.riscv64/hv6.bbl o.riscv64/hv6.bbl.img（将其转换为hv6.bbl.img）;
- 由于没有文件系统，所以，将ulib和init两个elf文件通过.incbin "init"和.incbin "ulib" 汇编指令(hv6/initbin.S)包含到整个内核镜像中。