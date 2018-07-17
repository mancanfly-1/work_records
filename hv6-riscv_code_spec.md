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
    
    current = Map(pid_t)                               	# 当前运行的进程
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

----
## syscall分析

### 进程管理代码分析
#### 进程数据结构和规约对比
![image](/images/Code_Proc.png)
![image](/home/zhangqiang/Pictures/Spec_Proc.png)<br>
两者重要的区别是hvm虽然是一个页，但在code中有详细的数据结构，这需要我们自己来维护。而x86中是由虚拟化技术来实现的。
#### sys_clone(pid_t pid, pn_t pml4, pn_t stack, pn_t hvm)(clone_proc):this is called by sys_clone in entry.S.
1. 根据给出的pml4, stack和hvm pages，调用alloc_proc方法来创建进程
2. 如果创建失败，退出；否则，获得current进程，将当前进程的stack拷贝到pid的stack中
3. 将current的hvm拷贝到pid的hvm里
4. 修改hvm中的关键信息,调用方法为hvm.c 中riscv_hvm_copy方法<br>
```c
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
	/* 如上所示，如果验证通过，则说明parent_hvm.tf-->gpr.a0 = 1， parent_hvm->tf =
	   parent_hvm->context.sp， parent_hvm->context.ra = */
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

----
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
```

```python
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

----
#### sys_reparent（reparent操作的含义是将当前pid进程的父亲进程指定为INITPID，pid为要reparent的进程id）
1. 判断pid是否合法，如果合法，获取pid对应的进程；否则退出
2. 判断pid对应的进程状态是否合法(非PROC_ZOMBIE),同时，INITPID进程的状态也不能为PROC_RUNNABLE和PROC_RUNNING两种状态
3. INITPID的nr_children++;pid对应的父进程的nr_children--
4. 设置pid进程的ppid为INITPID<br>
可以看出，上面的操作，只涉及到了两个进程的nr_children和ppid属性，在规约中很好对应，没有问题。

---
#### sys_set_runnable
该syscall的工作是将制定pid的进程状态设定为PROC_RUNNABLE状态。唯一要进行修改的进程属性是proc->state；<br>
修改该属性的前提：
1. 只有parent进程能够修改自己children进程的运行状态
2. 对于要修改进程的状态一定不能为PROC_EMBRYO
该syscall比较简单，状态空间变化不大，与代码一致。

---
### 内存相关syscall

#### sys_reclaim_page
对于回收page的主要工作是回收该page对应的page_descriptor. page里的内容可以不清零，只要标识它已经被回收就可以了。
在回收前，要确保该页的合法和该页的所属Proc合法
1. 页合法：该page的类型非PAGE_TEYP_FREE；(已经free了，还释放啥啊！)
2. Proc合法：该页所属进程的状态为PROC_ZOMBIE；同时，该进程没有使用任何设备，即Proc->nr_devs== 0.???device对应的page不是要释放的page不可以吗？
3. 释放时，该页的类型变为PAGE_TYPE_FREE；该页原来所属进程拥有的页数减一。
状态空间的改变涉及到了两个属性，desc_table和Proc->nr_pages
    
---
#### sys_alloc_pdpt
```c
int sys_alloc_pdpt(pid_t pid, pn_t from, size_t index, pn_t to, pte_t perm)
{
    return alloc_page_table_page(pid, from, index, to, perm & ~PTE_XWR_MASK, PAGE_TYPE_X86_PML4, PAGE_TYPE_X86_PDPT);
}
```
---
#### sys_alloc_pd
```c
int sys_alloc_pd(pid_t pid, pn_t from, size_t index, pn_t to, pte_t perm)
{
    return alloc_page_table_page(pid, from, index, to, perm & ~PTE_XWR_MASK, PAGE_TYPE_X86_PDPT, PAGE_TYPE_X86_PD);
}
```
---
#### sys_alloc_pt
```c
int sys_alloc_pt(pid_t pid, pn_t from, size_t index, pn_t to, pte_t perm)
{
    return alloc_page_table_page(pid, from, index, to, perm & ~PTE_XWR_MASK, PAGE_TYPE_X86_PD, PAGE_TYPE_X86_PT);
}
```
---
#### sys_alloc_frame
```c
int sys_alloc_frame(pid_t pid, pn_t from, size_t index, pn_t to, pte_t perm)
{
    return alloc_page_table_page(pid, from, index, to, perm, PAGE_TYPE_X86_PT, PAGE_TYPE_FRAME);
}
```
---
#### alloc_page_table_page
前面4个系统调用使用本共同方法来完成页表各个级别表的建立，根据传入的参数不同，创建的页表也不同。
```c
/* 在这里要保证from_type and to_type的四级页表上下级关系，还好这个函数在上面的
   四个syscalls中都写死了，确保了上下级关系。而该函数作为内部函数不能被外界调用。 
*/
static int alloc_page_table_page(pid_t pid, pn_t from_pn, size_t index, pn_t to_pn, pte_t perm,
                                 enum page_type from_type, enum page_type to_type)
{
    int r;
    pn_t pfn;
    /* to_pn must be PAGE_TYPE_FREE */
    if (!is_page_type(to_pn, PAGE_TYPE_FREE))
        return -EINVAL;
    // 页帧号= pages地址/page_size; pages数组不是从0开始计数的。
    pfn = pn_to_pfn(to_pn);
    
    r = map_page(pid, from_pn, index, pfn, perm, from_type);
    if (r)
        return r;
    /* alloc_page always succeeds so it's okay to call it after map_page */
    alloc_page(pid, to_type, to_pn);
    return 0;
}

/* 
    主要操作是通过pn获得对应的page_desc，设置该pn属于哪个pid，以及该页的类型
*/
void alloc_page(pid_t pid, enum page_type type, pn_t pn)
{
    struct page_desc *desc = get_page_desc(pn);

    assert(is_page_type(pn, PAGE_TYPE_FREE), "must be a free page");
    desc->pid = pid;
    desc->type = type;

    if (pn != 0)
        FREELIST_DEL(page_desc_table, link, pn);

    bzero(get_page(pn), PAGE_SIZE);
    // 不用担心判断pid失败，pid在调用该函数之前就已经判断了，在python中执行符号执行时，失败的分支是不会走的；
    if (pid)
        ++get_proc(pid)->nr_pages;
}
```
分析：
    syscall调用，要修改page，proc对应拥有page的数量，其中page中包括from_page的entry信息，以及from_page与to_page之间的对应关系，不同处在于，1. permmision基于riscv，10位标识位。2. 各参数名称不同。
    在这里有个地方需要注意，spec中page结构体定义了很多辅助信息，如单独的perm、shadow page等，这些信息在程序中是没有保存的，目的是为了验证上层属性的方便。在equal.py中的page_equal只针对entry[index] 相等。
```python
util.global_to_uf_dict(ctx, '@pages')[()](util.i64(0), pn, idx) == kernelstate.pages[pn].data(idx)
```
#### sys_copy_frame
1. 首先判断from和to两个pf的合法性；
2. 直接调用memcpy就可以了，只是page中相应的index值发生了变化，状态空间变化很小。
OK
---
#### sys_protect_frame(该syscall的作用是设置指定pte index的permission)
1. 判断要设置的pte对应的page table、process、frame满足条件
2. 判断pte的合法性
3. 检查perm的合法性
4. 更新状态空间,只是更新了指定pte的值，并更新tlb。防止tlb命中后，造成的不一致。还有一个不同是通过下面代码获得pfn时，由于结构不同，PAGE_SHIFT由x86 12位--》10位
```c
pfn = PTE_ADDR(entries[index]) >> PAGE_SHIFT;
```
这段代码
```c
entries[index] = (pfn << PTE_PFN_SHIFT) | perm;
hvm_invalidate_tlb(current);
```
说明：PTE_PFN_SHIFT与x86是不一致的，10位，在x86中是12位。在判断的过程中，要求对应page table的type是PAGE_TYPE_X86_PT, 这也应该修改一下，毕竟是在riscv下。规约中关于PAGE_SHIFT的判断也存在不同。

---
#### sys_map_proc（将proc结构体所在的page映射到指定页上）
```c
/* 参数n代表要映射的proc页*/
int sys_map_proc(pid_t pid, pn_t from, size_t index, size_t n, pte_t perm)
{
    pn_t pfn;
    // 要映射的页号不能超出proc数组所在内存的范围
    if (n >= bytes_to_pages(NPROC * sizeof(struct proc)))
        return -EINVAL;
    pfn = (uintptr_t)proc_table / PAGE_SIZE + n;
    //perm must writable
    if (pte_writable(perm))
        return -EACCES;
    return map_page(pid, from, index, pfn, perm, PAGE_TYPE_X86_PT);
}
```
说明：状态空间只改了要映射的page对应from page的pte，状态空间变化较小。记住map_page一定要flush tlb。

---
#### sys_map_pml4(将proc->page_table_root page frame映射到本身指定的index上)
该方法同样调用了map_page
其中pid, index, perm是传入的参数
from：proc->page_table_root;
pfn: from的frame number
即：设置page_table_root中index位置entry的值。
```c
return map_page(pid, from, index, pfn, perm, PAGE_TYPE_X86_PML4);
```
---
#### sys_map_page_desc（将page_desc_table对应的某一个页映射到页表中。）
```c
return map_page(pid, from, index, pfn, perm, PAGE_TYPE_X86_PT);
```
---
#### sys_map_dev

```c
return map_page(pid, from, index, pfn, perm, PAGE_TYPE_X86_PT);
```
---
#### sys_map_file

```c
return map_page(pid, from, index, pfn, perm, PAGE_TYPE_X86_PT);
```
---
#### map_page
在上面sys_alloc_xxx 四个syscalls的前三个perm = perm & ~PTE_XWR_MASK, 同时，4个sys_map_xxx在最后也是调用了这个函数
该函数对状态空间的pages相关信息产生影响；同时，页表的更变也必须刷tlb。总体来说，上面的几个syscall与x86相比，改变的主要为perm信息。
```c
/* 首先要判断map_page参数的合法性，不用判断from_type和to_type的上下级关系。 map_page的主要工作是设置from_pn的index位置的entry值，
    即，entries[index] = (pfn << PTE_PFN_SHIFT | perm); 页帧号 + permission perm中的XER位在非叶子页表的时候，必须为0，*/
int map_page(pid_t pid, pn_t from_pn, size_t index, pn_t pfn, pte_t perm, enum page_type from_type)
{
    pte_t *entries;
    if (!is_pid_valid(pid))
        return -ESRCH;
    /* check if pid is current or its embryo */
    if (!is_current_or_embryo(pid))
        return -EACCES;
    if (!is_page_type(from_pn, from_type))
        return -EINVAL;
    /* check if pid owns from_pfn */
    if (!is_page_pid(from_pn, pid))
        return -EACCES;
    if (!is_page_index_valid(index))
        return -EINVAL;
    /* no check on pfn; left to caller */
    /* check for unsafe bits in page permissions */
    if (perm & ~PTE_PERM_MASK)
        return -EINVAL;
    /* make sure we have non-zero entries */
    if (!pte_valid(perm))
        return -EINVAL;

    entries = get_page(from_pn);
    /* make sure the entry is empty; may not be necessary but good to check */
    if (pte_valid(entries[index]))
        return -EINVAL;

    /* update the page table */
    mmio_write64(&entries[index], (pfn << PTE_PFN_SHIFT) | perm);
    hvm_invalidate_tlb(pid);
    return 0;
}
```
---
#### sys_free_pdpt
```c
return free_page_table_page(from, index, to, PAGE_TYPE_X86_PML4, PAGE_TYPE_X86_PDPT);
```

---
#### sys_free_pd
```c
return free_page_table_page(from, index, to, PAGE_TYPE_X86_PDPT, PAGE_TYPE_X86_PD);
```

---
#### sys_free_pt
```c
return free_page_table_page(from, index, to, PAGE_TYPE_X86_PD, PAGE_TYPE_X86_PT);
```
---
#### sys_free_frame
```c
return free_page_table_page(from, index, to, PAGE_TYPE_X86_PT, PAGE_TYPE_FRAME);
```
---

#### free_page_table_page
内部定义函数，sys_free_xxxx的四个syscall在最后都调用了这个共同方法；与x86_64相比，riscv无变化。与原来的spec一致。
```c
/*
 * We need to_pn as a parameter; otherwise it's hard to check
 * if a given entry has a valid pn.
 */
static int free_page_table_page(pn_t from_pn, size_t index, pn_t to_pn, enum page_type from_type,
                                enum page_type to_type)
{
    pte_t *entries;

    if (!is_page_type(from_pn, from_type))
        return -EINVAL;
    if (!is_page_pid(from_pn, current))
        return -EACCES;
    if (!is_page_index_valid(index))
        return -EINVAL;
    if (!is_page_type(to_pn, to_type))
        return -EINVAL;
    if (!is_page_pid(to_pn, current))
        return -EACCES;

    entries = get_page(from_pn);
    /* check if the slot is empty */
    if (!pte_valid(entries[index]))
        return -EINVAL;
    /* check if the entry matches */
    if (pn_to_pfn(to_pn) != (PTE_ADDR(entries[index]) >> PAGE_SHIFT))
        return -EINVAL;

    /* release the next-level page, free page的操作是将该页对应的page_desc->pid page_desc->type都清空
        并将该页对应的进程拥有的页数量减1. */
    free_page(to_pn);
    /* wipe the entry */
    entries[index] = 0;
    hvm_invalidate_tlb(current);

    return 0;
}
```
#### sys_map_page(这是新添加的syscall，应用：map_page方法作为系统调用用来被外面使用，因为对于将内核代码建立页表时，需要该方法)
将map_page作为系统调用，向外暴露，目的是在调用setop_kernel_image时，可以将整个内核映射到页表中。

----

### 文件系统调用接口

### 设备相关系统调用接口
在RISCV中不涉及IOMMU，所以相关调用接口都需要去掉，相关接口包括：<br>
sys_alloc_iommu_root<br>
sys_alloc_iommu_pdpt<br>
sys_alloc_iommu_pd<br>
sys_alloc_iommu_pt<br>
sys_alloc_iommu_frame<br>
sys_map_iommu_frame<br>
sys_reclaim_iommu_frame<br>
sys_reclaim_iommu_root<br>

#### sys_map_pcipage
映射pcipage的方法与映射其他页相同，但主要判断pcipage的合法性和该页是否属于要映射的进程

----
####  sys_alloc_vector
设置vector_table中该对应设备的进程为当前进程，即当前进程拥有该vector，x86与riscv状态空间没有任何变化

----

#### sys_reclaim_vector
回收vector_table中相应vector，即设置vector_table中的vector等于0.并减少该进程对应的vector数。
需要注意只有非PROC_ZOMBIE状态的进程并且该进程的nr_intremaps数量大于0时才能够回收vector。
与riscv相比没有任何状态空间的变化。所以，也不用修改syscall对应的代码。

----
#### sys_alloc_intremap
alloc一个intremap
#### sys_reclaim_intremap
回收一个intremap
#### sys_ack_intr
设置进程的intr属性
#### device相关syscalls中vector_table、pci_table、intremap_table之间的关系？
vector_table: 存储的是指定vector对应的pid<br>
pci_table: devid --> pid<br>
intremap_table: index --> intremap<br>

```c
// intremap结构体
struct intremap {
    enum intremap_state state;
    devid_t devid;
    uint8_t vector;
};
```

### 文件系统相关函数
文件syscalls在fd.c中定义和实现。riscv中并没有对这部分内容进行任何更改。<br>
sys_create<br>
sys_close<br>
sys_dup<br>
sys_dup2<br>
sys_lseek<br>

### IPC
IPC相关syscalls定义在ipc.c中，riscv中并没有对这部分内容进行任何更改<br>
sys_send<br>
sys_recv<br>
send_recv<br>
sys_reply_wait<br>
sys_call<br>

### IO
IO相关系统调用在ioport.c中定义和实现，riscv中并没有对这部分内容进行任何更改<br>
sys_alloc_io_bitmap<br>
sys_alloc_port<br>
sys_reclaim_port<br>
----
### 总结
    由于riscv与x86 page_table_entry的不同syscalls中主要修改了page_table的各个标识位。
- entry的结构发生了变化，PTE_PFN_SHIFT由原来的12位变为10位；
- 各标识位改变；
- 非叶子page_table节点中的标识位发生变化：PTE_XWR_MASK = PTE_W | PTE_X | PTE_R，这几个位必须为0；
- riscv中更改了hvm，用于保存当前context和trap帧。这部分内容修改了程序，但在规约中没有体现；
- 新添加了sys_map_page方法用于对内核代码进行映射。

### 学习过程中查询了相关信息
- 程序在编译过程中的链接地址是物理地址还是虚拟地址？还是都不是？
    - 运行地址《--》链接地址
    - 存储地址《--》加载地址
    - 程序的链接地址<br>
    显然是虚拟地址。在ulib.lds中使用的是0xffffffff80000000，但是kernel加载的位置0x80000000的虚拟地址等于物理地址。
- 编译过程<br>
    1. 首先根据hv6/user/lib/ulib.lds，链接成ulib;
    2. 然后生成init用户程序elf;(没有指定链接脚本，所以，从0x0作为链接地址(即，运行地址))
    3. 根据kernel.lds，将hv6的内核各个目标文件链接成hv6.elf.链接地址是0x80000000。为什么使用这个地址呢？<br>
    张蔚的回答：目前该地址对应的就是物理地址。如果不使用这个地址，就需要bbl为我们提供地址的映射方案，但目前由于bbl当前版本已经不提供了，所以直接使用了这个地址，即物理地址。
    4. 使用objcopy方法将hv6.elf转换成hv6.bbl.dbg --adjust-vma 0x80000000（修改了链接地址）;
    5. cp bbl ../../hv6/o.riscv64/hv6.bbl（首先复制bbl为hv6.bbl）;
    6. riscv64-unknown-elf-objcopy -O binary o.riscv64/hv6.bbl o.riscv64/hv6.bbl.img（将其转换为hv6.bbl.img）;
- 由于没有文件系统，所以，将ulib和init两个elf文件通过.incbin "init"和.incbin "ulib" 汇编指令(hv6/initbin.S)包含到整个内核镜像中。
