# Lab 3 page tables

[TOC]

​	本实验将探索页表并对其进行修改，以简化将数据从用户空间复制到内核空间的函数。

## 1. Print a page table ([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

### 实验目的

​	使用页表机制加速系统调用：一些操作系统（如 Linux）通过在用户空间和内核之间共享只读区域中的 数据来加快某些系统调用的速度。操作系统不会将内核的数据结构复制到用户空间，而是直接将存放该 数据结构的页通过页表 的机制映射到用户空间的对应位置，从而免去了复制的开销，提高了系统的性 能。本实验旨在学习如何在页表中插入映射，首先需要在 xv6 中的 getpid() 系统调用中实现这一优 化。 

​	通过在用户空间和内核之间共享一个只读区域中的数据，来加速特定的系统调用。具体而言，通过在进 程创建时映射一个只读页，将一个 struct usyscall 结构放置在该页的开头，该结构会存储当前进程 的 PID 。这将使得在执行某些系统调用时，不需要进行用户空间和内核之间的频繁切换，从而提高系 统调用的性能。



### 实验过程

1. 在`kernel/defs.h`中定义函数原型并在`kernel/exec.c`调用：

   在`exec.c`中的`return argc`之前插入`if(p->pid==1) vmprint(p->pagetable)`

```c
//kernel/def.h
//...
// exec.c
int             exec(char*, char**);
void            vmprint(pagetable_t);
//...

//kernel/exec.c
//...
if(p->pid==1)
    vmprint(p->pagetable);
return argc;
```

2. 研究`kernel/vm.c`中的`freewalk`函数，在`freewalk`函数中，首先会遍历整个`pagetable`，当遍历到一个有效的页表项且该页表项不是最后一层时，会`freewalk`函数会递归调用。`freewalk`函数通过`(pte & PTE_V)`判断页表是否有效，`(pte & (PTE_R|PTE_W|PTE_X)) == 0`判断是否在不在最后一层。参考`freewalk`函数，我们可以在`kernel/vm.c`中写出`vmprint`函数的递归实现，代码如下：

```c
void
vmprint_rec(pagetable_t pagetable, int level){
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    // PTE_V is a flag for whether the page table is valid
    if(pte & PTE_V){
      for (int j = 0; j < level; j++) {
        if (j) 
          printf(" ");
        printf("..");
      }
      uint64 child = PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, child);
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0) {
        // this PTE points to a lower-level page table.
        vmprint_rec((pagetable_t)child, level + 1);
      }
    }
  }
}

void
vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  vmprint_rec(pagetable, 1);
}
```

​	上述代码中，`vmprint_rec`是递归函数，`level`传入层级来根据层级数量输出".."，`vmprint`是真正的输出函数，通过调用递归函数`vmprint_rec`实现。

3. 测试

输入`make qemu`，发现有如下输出结果：

```shell
xv6 kernel is booting

hart 2 starting
hart 1 starting
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
init: starting sh
```

说明个人测试成功，下面进行单元测试，输出如下结果：

```shell
bronya_k@LAPTOP-TBUJAUQE:~/xv6-labs-2020-3/xv6-labs-2020$ ./grade-lab-pgtbl pte print
make: 'kernel/kernel' is up to date.
== Test pte printout == pte printout: OK (0.9s)
```

说明单元测试成功。

### 实验中遇到的问题和解决办法

 1.问题：未正确在用户页表中添加共享页的映射。 

- 解决办法：在 proc_pagetable() 函数中使用 mappages() 函数添加共享页的映射，并设置相 应的标记位。

2.问题：如果权限设置不正确，可能导致用户空间尝试写入只读页，导致错误或崩溃。如 proc_pagetable(struct proc *p) 中映射 PTE 时的权限应该为 PTE_R | PTE_U 而不是 PTE_R | PTE_U | PTE_W 。 

- 解决办法：用户程序不应该直接写入代码或数据段，这是由操作系统负责的。因此，在将这些代码 和数据映射到用户页表时，不应该使用可写权限。

### 实验心得

1. 通过将系统调用的相关数据放在只读页中，以减少内核和用户空间之间的数据传输次数，从而加速 系统调用的执行,体现了性能优化。 通过在每个进程的页表中插入只读页，掌握操作页表的方法，从而实现用户空间与内核空间之间的 数据共享。 通过这次实验，了解了如何在页表中插入 PTE ，以加速特定系统调用。

## 2.A kernel page table per process ([hard](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

### 实验目的

​	打印页表：深入理解 RISC-V 页表的结构和内容，并提供一个接受 pagetable_t 参数，打印页表的函数 vmprint() 。通过这个实验，实现可视化页表的布局，了解页表的层次结构以及如何将虚拟地址映射 到物理地址

### 实验过程

1. 按照提示，先在`kernel/proc.h `的`struct proc` 结构体的中添加`kernalpagetable`即内核页表字段：

```c
struct proc {
  //...
  pagetable_t pagetable;       // User page table
  pagetable_t kernelpagetable;       // kernel page table
  //...
};
```

2. 初始化内核页表`kernelpagetable`。可以模仿`kernel/vm.c`里的`kvminit`函数，用内核自己`pagetable `初始化的方式初始化用户进程的内核页表，模仿`kvmmap()`函数在`kernel/vm.c`新增`uvmmap()`函数为用户进程的内核页表添加映射，方便初始化内核页表`uvmKernelInit`函数的编写。

```c
void uvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm) {
    if(mappages(pagetable, va, sz, pa, perm) != 0) {
        panic("uvmmap");
    }
}

pagetable_t uvmKernelInit(struct proc *p) {
    pagetable_t kpagetable = uvmcreate();
    if(kpagetable == 0){
        return 0;
    }

    uvmmap(kpagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);
    uvmmap(kpagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
    uvmmap(kpagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
    uvmmap(kpagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
    uvmmap(kpagetable, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
    uvmmap(kpagetable, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
    uvmmap(kpagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);    // 注意va为TRAMPOLINE

    return kpagetable;
}
```

3. 在`defs.h`中声明`uvmKernelInit()`后在`kernel/proc.c`中的`allocproc`中调用`uvmKernelInit()`函数以实现用户进程内核页表的初始化。

```c
//kernel/defs.h
// vm.c
//...
pagetable_t     uvmKernelInit(struct proc *p)
    
//kernel/proc.c
static struct proc*
allocproc(void)
{
  //...
  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  //add code to call user kernel pagetable
  p->kernelpagetable = uvmKernelInit(p);
  if (p->kernelpagetable == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  } 
  //...
}
```

3. 提示要求每个内核页表中都添加一个对自己内核栈的映射，未修改前，内核栈初始化在`kernel/proc.c`的`procinit`函数中，将`stack`初始化的部分移入`kernel/proc.c`的`allocproc`函数中，完成对`allocproc`函数的修改。

```c
void
procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");

      // Allocate a page for the process's kernel stack.
      // Map it high in memory, followed by an invalid
      // guard page.
      //char *pa = kalloc();
      //if(pa == 0)
      //  panic("kalloc");
      //uint64 va = KSTACK((int) (p - proc));
      //kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
      //p->kstack = va;
  }
  kvminithart();
}

static struct proc*
allocproc(void)
{
//...
// Allocate a page for the process's kernel stack
  char *pa = kalloc();
      if(pa == 0)
        panic("kalloc");
      uint64 va = KSTACK((int) (p - proc));
      kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
      p->kstack = va;
  // mapping virtual address and physical address
  uvmmap(p->kernelpagetable,va, (uint64)pa,PGSIZE,PTE_R | PTE_W);
  p->kstack = va;


  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
//... 
```

4. 参考`kvminithart`函数，修改`proc.c`里的`scheduler`函数，在进程进行切换的时候把自己的内核页表放入到`stap`寄存器里，让内核使用该进程的页表转换地址，如果没有进程在运行，就使用内核自己的内核页表。`stap`寄存器负责管理内核页面。修改后的`scheduler`代码如下：

```c
void
scheduler(void)
{
  //...
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        // load the process's kernel page table - lab3-2
        w_satp(MAKE_SATP(p->kernelpagetable));
		// flush TLB
        sfence_vma();
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```

5. 进程销毁部分添加对内核页表的释放，由于之前将`kernel stack`的初始化移入了`allocproc`函数中，所以先要对kernel stack进行查找释放，代码如下：

```c
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  // free kernel stack
  if (p->kstack)
  {
    pte_t* pte = walk(p->kernelpagetable, p->kstack, 0);
    if (pte == 0)
      panic("freeproc: kstack");
    kfree((void*)PTE2PA(*pte));
  }
  p->kstack = 0;
  //...
```

然后需要释放用户进程的内核页表：

```c
static void
freeproc(struct proc *p)
{
  //...
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  // free kernel pagetable
  if (p->kernelpagetable)
    proc_freeKernelPageTable(p->kernelpagetable);
  //...
}
```

6. 需要保证释放页表的时候而无需同时释放叶节点的物理内存页，所以需要添加一个`proc_freeKernelPageTable`特殊处理内存页表的释放以保证这一点。该函数代码如下：

```c
void 
proc_freeKernelPageTable(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V)){
      pagetable[i] = 0;
      if ((pte & (PTE_R|PTE_W|PTE_X)) == 0)
      {
        uint64 child = PTE2PA(pte);
        proc_freeKernelPageTable((pagetable_t)child);
      }
    } else if(pte & PTE_V){
      panic("proc free kpt: leaf");
    }
  }
  kfree((void*)pagetable);
}
```

​	最后根据启动`make qemu`时产生的报错将不在`defs.h`中声明过的函数原型添加进去，此时启动`make qemu`输出如下：

```shell
xv6 kernel is booting

hart 2 starting
hart 1 starting
panic: kvmpa
```

​	这是由于` virtio` 磁盘驱动 `virtio_disk.c` 中的`virtio_disk_rw()`函数调用了 `kvmpa()` 用于将虚拟地址转换为物理地址，此时需要换为当前进程的内核页表。所以我们需要将 `kvmpa()` 的参数增加一个 `pagetable_t`, 在 `virtio_disk_rw()`函数调用时传入 `myproc()->kernelpagetable`.为了保证`virtio_disk.c`在调用时不报错，需要给`virtio_disk.c`添加头文件引用`#include "proc.h"`。修改后的`kvmpa()`函数如下：

```c
uint64
kvmpa(pagetable_t kernelPageTable, uint64 va)
{
  uint64 off = va % PGSIZE;
  pte_t *pte;
  uint64 pa;
  pte = walk(kernelPageTable, va, 0);
  if(pte == 0)
    panic("kvmpa");
  if((*pte & PTE_V) == 0)
    panic("kvmpa");
  pa = PTE2PA(*pte);
  return pa+off;
}
```

​	再次键入`make qemu`发现可以正常启动。

7. 测试，需要花费较长时间，启动`qemu`后运行`usertest`，输出结果如下，说明正确：

```shell
init: starting sh
$ usertests
usertests starting
test execout: OK
test copyin: OK
test copyout: OK
test copyinstr1: OK
test copyinstr2: OK
test copyinstr3: OK
test truncate1: OK
test truncate2: OK
test truncate3: OK
test reparent2: OK
test pgbug: OK
test sbrkbugs: usertrap(): unexpected scause 0x000000000000000c pid=3234
            sepc=0x0000000000005406 stval=0x0000000000005406
usertrap(): unexpected scause 0x000000000000000c pid=3235
            sepc=0x0000000000005406 stval=0x0000000000005406
OK
test badarg: OK
test reparent: OK
test twochildren: OK
test forkfork: OK
test forkforkfork: OK
test argptest: OK
test createdelete: OK
test linkunlink: OK
test linktest: OK
test unlinkread: OK
test concreate: OK
test subdir: OK
test fourfiles: OK
test sharedfd: OK
test exectest: OK
test bigargtest: OK
test bigwrite: OK
test bsstest: OK
test sbrkbasic: OK
test sbrkmuch: OK
test kernmem: usertrap(): unexpected scause 0x000000000000000d pid=6214
            sepc=0x000000000000201a stval=0x0000000080000000
usertrap(): unexpected scause 0x000000000000000d pid=6215
            sepc=0x000000000000201a stval=0x000000008000c350
usertrap(): unexpected scause 0x000000000000000d pid=6216
            sepc=0x000000000000201a stval=0x00000000800186a0
usertrap(): unexpected scause 0x000000000000000d pid=6217
            sepc=0x000000000000201a stval=0x00000000800249f0
usertrap(): unexpected scause 0x000000000000000d pid=6218
            sepc=0x000000000000201a stval=0x0000000080030d40
usertrap(): unexpected scause 0x000000000000000d pid=6219
            sepc=0x000000000000201a stval=0x000000008003d090
usertrap(): unexpected scause 0x000000000000000d pid=6220
            sepc=0x000000000000201a stval=0x00000000800493e0
usertrap(): unexpected scause 0x000000000000000d pid=6221
            sepc=0x000000000000201a stval=0x0000000080055730
usertrap(): unexpected scause 0x000000000000000d pid=6222
            sepc=0x000000000000201a stval=0x0000000080061a80
usertrap(): unexpected scause 0x000000000000000d pid=6223
            sepc=0x000000000000201a stval=0x000000008006ddd0
usertrap(): unexpected scause 0x000000000000000d pid=6224
            sepc=0x000000000000201a stval=0x000000008007a120
usertrap(): unexpected scause 0x000000000000000d pid=6225
            sepc=0x000000000000201a stval=0x0000000080086470
usertrap(): unexpected scause 0x000000000000000d pid=6226
            sepc=0x000000000000201a stval=0x00000000800927c0
usertrap(): unexpected scause 0x000000000000000d pid=6227
            sepc=0x000000000000201a stval=0x000000008009eb10
usertrap(): unexpected scause 0x000000000000000d pid=6228
            sepc=0x000000000000201a stval=0x00000000800aae60
usertrap(): unexpected scause 0x000000000000000d pid=6229
            sepc=0x000000000000201a stval=0x00000000800b71b0
usertrap(): unexpected scause 0x000000000000000d pid=6230
            sepc=0x000000000000201a stval=0x00000000800c3500
usertrap(): unexpected scause 0x000000000000000d pid=6231
            sepc=0x000000000000201a stval=0x00000000800cf850
usertrap(): unexpected scause 0x000000000000000d pid=6232
            sepc=0x000000000000201a stval=0x00000000800dbba0
usertrap(): unexpected scause 0x000000000000000d pid=6233
            sepc=0x000000000000201a stval=0x00000000800e7ef0
usertrap(): unexpected scause 0x000000000000000d pid=6234
            sepc=0x000000000000201a stval=0x00000000800f4240
usertrap(): unexpected scause 0x000000000000000d pid=6235
            sepc=0x000000000000201a stval=0x0000000080100590
usertrap(): unexpected scause 0x000000000000000d pid=6236
            sepc=0x000000000000201a stval=0x000000008010c8e0
usertrap(): unexpected scause 0x000000000000000d pid=6237
            sepc=0x000000000000201a stval=0x0000000080118c30
usertrap(): unexpected scause 0x000000000000000d pid=6238
            sepc=0x000000000000201a stval=0x0000000080124f80
usertrap(): unexpected scause 0x000000000000000d pid=6239
            sepc=0x000000000000201a stval=0x00000000801312d0
usertrap(): unexpected scause 0x000000000000000d pid=6240
            sepc=0x000000000000201a stval=0x000000008013d620
usertrap(): unexpected scause 0x000000000000000d pid=6241
            sepc=0x000000000000201a stval=0x0000000080149970
usertrap(): unexpected scause 0x000000000000000d pid=6242
            sepc=0x000000000000201a stval=0x0000000080155cc0
usertrap(): unexpected scause 0x000000000000000d pid=6243
            sepc=0x000000000000201a stval=0x0000000080162010
usertrap(): unexpected scause 0x000000000000000d pid=6244
            sepc=0x000000000000201a stval=0x000000008016e360
usertrap(): unexpected scause 0x000000000000000d pid=6245
            sepc=0x000000000000201a stval=0x000000008017a6b0
usertrap(): unexpected scause 0x000000000000000d pid=6246
            sepc=0x000000000000201a stval=0x0000000080186a00
usertrap(): unexpected scause 0x000000000000000d pid=6247
            sepc=0x000000000000201a stval=0x0000000080192d50
usertrap(): unexpected scause 0x000000000000000d pid=6248
            sepc=0x000000000000201a stval=0x000000008019f0a0
usertrap(): unexpected scause 0x000000000000000d pid=6249
            sepc=0x000000000000201a stval=0x00000000801ab3f0
usertrap(): unexpected scause 0x000000000000000d pid=6250
            sepc=0x000000000000201a stval=0x00000000801b7740
usertrap(): unexpected scause 0x000000000000000d pid=6251
            sepc=0x000000000000201a stval=0x00000000801c3a90
usertrap(): unexpected scause 0x000000000000000d pid=6252
            sepc=0x000000000000201a stval=0x00000000801cfde0
usertrap(): unexpected scause 0x000000000000000d pid=6253
            sepc=0x000000000000201a stval=0x00000000801dc130
OK
test sbrkfail: usertrap(): unexpected scause 0x000000000000000d pid=6265
            sepc=0x0000000000003e7a stval=0x0000000000012000
OK
test sbrkarg: OK
test validatetest: OK
test stacktest: usertrap(): unexpected scause 0x000000000000000d pid=6269
            sepc=0x0000000000002188 stval=0x000000000000fbc0
OK
test opentest: OK
test writetest: OK
test writebig: OK
test createtest: OK
test openiput: OK
test exitiput: OK
test iput: OK
test mem: OK
test pipe1: OK
test preempt: kill... wait... OK
test exitwait: OK
test rmdot: OK
test fourteen: OK
test bigfile: OK
test dirfile: OK
test iref: OK
test forktest: OK
test bigdir: OK
ALL TESTS PASSED
```



### 实验中遇到的问题和解决办法

​	该实验步骤繁琐，要注意的点比较多，需要重新学习的知识也很多，在具体的编程过程中遇到的问题主要是在最后一步修改`kvmpa`函数后报错如下：

```c
kernel/virtio_disk.c: In function 'virtio_disk_rw':
kernel/virtio_disk.c:206:51: error: dereferencing pointer to incomplete type 'struct proc'
  206 |   disk.desc[idx[0]].addr = (uint64) kvmpa(myproc()->kernelpagetable, (uint64) &buf0);
      |                                                   ^~
make: *** [<builtin>: kernel/virtio_disk.o] Error 1
```

​	这是由于`virtio_disk.c`文件在编译时无法找到`struct proc`结构体所致，该结构体定义在`proc.h`文件中，于是在`virtio_disk.c`中添加头文件引用`proc.h`即可。

​	内核栈页面：每个进程都有自己的内核栈，它将映射到偏高一些的地址，这样xv6在它之下就可以留下一个未映射的保护页(guard page)。保护页的PTE是无效的（也就是说`PTE_V`没有设置），所以如果内核溢出内核栈就会引发一个异常，内核触发`panic`。如果没有保护页，栈溢出将会覆盖其他内核内存，引发错误操作。panic crash是更可取的方案。

### 实验心得

**内核地址空间**：Xv6为每个进程维护一个页表，用以描述每个进程的用户地址空间，外加一个单独描述内核地址空间的页表。内核配置其地址空间的布局，以允许自己以可预测的虚拟地址访问物理内存和各种硬件资源。下图显示了这种布局如何将内核虚拟地址映射到物理地址。文件`kernel/memlayout.h`声明了xv6内核内存布局的常量。

![](http://xv6.dgs.zone/tranlate_books/book-riscv-rev1/images/c3/p3.png)

**`kvmpa`函数**：

​		在 xv6 操作系统中，`kvmpa` 函数用于将内核虚拟地址转换为物理地址。它是用于内核级别的虚拟地址到物理地址的映射转换。该函数的作用是将给定的虚拟地址 `va` 转换为内核页表 `kernelPageTable` 中对应的物理地址。修改后`kvmpa`函数的工作流程如下：

函数的工作流程如下：

1. 计算虚拟地址 `va` 在页面内的偏移量 `off`（即 `va % PGSIZE`）。
2. 调用 `walk` 函数，根据给定的内核页表 `kernelPageTable` 和虚拟地址 `va` 找到对应的页表条目（`pte`）。
3. 检查页表条目是否存在，如果为 `NULL`，即找不到对应的页表条目，触发 panic。
4. 检查页表条目是否有效（`PTE_V` 标志位是否被设置），如果无效，也触发 panic。
5. 将页表条目转换为物理地址（通过 `PTE2PA` 宏），并保存在变量 `pa` 中。
6. 将物理地址 `pa` 和页面内偏移量 `off` 相加，得到最终的物理地址，并将其作为函数的返回值。

修改后的`kvmpa`函数原码见上文。

## 3. Simplify `copyin/copyinstr` ([hard](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

### 实验目的

​	通过本次实验，我学会了如何使用用户空间缓冲区和内核空间缓冲区之间的数据传输。 实现 pgaccess() 系统调用的过程中，我深入了解了 RISC-V 页表中的访问位 PTE_A，并学会了如何 在内核中清除位。 通过实验初步了解了 RISC-V 中最常用的 Sv-39 （Page-Based 39-bit Virtual-Memory System ） 分页机制

### 实验过程

1. 通过调用定义在`kernel/vmcopyin.c`中的函数`copyin_new`和`copyinstr_new`来替换`copyin`和`copyinstr`函数，并在`defs.h`中添加`vmcopyin.c`文件中被用到的函数的声明。

2. 在`vm.c`中添加`vmcopypage()`函数，并将该函数声明添加到`defs.h`中。这个函数将进程中用户页表复制到内核页表中，代码如下：

3. 在`sysproc.c`修改和`fork()`，`sbrk()`，`exec()`有关的函数的内容，初次之外，还需要修改`userinit`函数和`exec.c`中的`exec`函数，在他们中调用`vmcopypage`函数，将进程页表复制到进程的内核页表中，修改后的代码如下：

4.测试：触发了`panic: kerneltrap`，出错，但不知道什么问题，报错如下：



## 提交



```shell
== Test pte printout ==
$ make qemu-gdb
pte printout: OK (5.5s)
== Test answers-pgtbl.txt == answers-pgtbl.txt: FAIL
    Cannot read answers-pgtbl.txt
== Test count copyin ==
$ make qemu-gdb
Timeout! count copyin: FAIL (150.3s)
    got:
      1
    expected:
      2
    QEMU output saved to xv6.out.count
== Test usertests ==
$ make qemu-gdb
Timeout! (300.3s)
== Test   usertests: copyin ==
  usertests: copyin: OK
== Test   usertests: copyinstr1 ==
  usertests: copyinstr1: OK
== Test   usertests: copyinstr2 ==
  usertests: copyinstr2: OK
== Test   usertests: copyinstr3 ==
  usertests: copyinstr3: OK
== Test   usertests: sbrkmuch ==
  usertests: sbrkmuch: FAIL
    Failed sbrkmuch
== Test   usertests: all tests ==
  usertests: all tests: FAIL
    ...
         test exectest: OK
         test bigargtest: scause 0x000000000000000d
         sepc=0x00000000800065aa stval=0x0000000000006778
         panic: kerneltrap
         qemu-system-riscv64: terminating on signal 15 from pid 910 (make)
    MISSING '^ALL TESTS PASSED$'
== Test time ==
time: OK
Score: 31/66
make: *** [Makefile:316: grade] Error 1
```

