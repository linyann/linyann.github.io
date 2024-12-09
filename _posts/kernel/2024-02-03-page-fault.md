---
layout: post
category: kernel
title: "缺页异常"
date: 2024-02-03 15:00:00 +0800
---

## 发生缺页异常的场景有哪些

1. 首次访问申请的虚拟地址空间，此时虚拟内存和物理内存没有建立映射关系。
2. 地址的映射关系已经建立，但是物理内存已经被换页到磁盘上。

## 缺页异常如何处理

函数调用：`do_page_fault -> handle_pte_fault`

参考：<https://zhuanlan.zhihu.com/p/195580742?utm_source=wechat_timeline>

![img](https://github.com/Geass-LL/draw/raw/master/github-io/page-fault.webp)

* 阶段1：判断缺页异常是否发生在内核线程或原子上下文中(中断也属于一种原子上下文)，是的话执行do_kernel_fault尝试修复或报段错误。
* 阶段2：判断是否是内核态访问用户地址空间的情况，是的话判断是否是指定的三种情况，是则报段错误。
* 阶段3：进入_do_page_fault, 查找异常地址所在的vm_area_struct域，并走表(page table walk)查找address对应PGD PUD PMD，最终* 找到PTE。
* 阶段4：进入handle_pte_fault()，判断PTE为空的话，说明用户空间申请了虚拟地址后第一次访问，尚未映射物理页面。在根据页面类型分别执行do_anonymous_page或do_fault。
* 阶段5：PTE非空表示已经建立过映射。判断PTE的present位是否为真，非真说明页面被swapping到磁盘上，随即执行do_swap_page。
* 阶段6：判断PTE_PROT_NONE是否为真，若为真执行do_numa_page产生页面迁移。
* 阶段7：判断错误类型，若是写类型的错误，再判断PTE的读写权限。只读的话说明页面是写保护的，调用do_wp_page。
* 阶段8: 页面可读可写仍然缺页异常，说明PTE_AF bit位是0，通过mk_young将其置1，表明此页面最近有被访问(页面回收中的第二次机会法)。 同时兼容ARM32，ARM32体系架构的Hardware PTE中不支持DIRTY ACCESS等bit位，所以通过软件配合缺页异常进行模拟。

do_page_fault的入参是发生异常时使用的寄存器集合，提供异常原因的错误码信息。发生缺页异常的地址可以从寄存器中读取，x86读取的是CR2寄存器。

硬件识别缺页异常中断，并找到对应的中断处理函数，以x86为例：

1. idt.c: INTG(X86_TRAP_PF, page_fault)
2. entry_64.S: idtentry page_fault do_page_fault
3. fault.c: do_page_fault

## 换页

* 换页：当物理内存容量不够的时候，操作系统把若干物理页的内容写到磁盘，回收物理页继续使用。换页时，会在页表中去除虚拟页的映射，同时记录这个物理页在磁盘的位置。
* 缺页异常：当应用程序访问已经分配但是没有映射到物理内存的虚拟页时就会触发缺页异常，操作系统会执行预先设置的缺页异常处理函数，这个函数会找到一个空闲的物理页，把之前保存在磁盘上的数据内容重新加载到物理页中。

## 几种不同的pagefault

参考维基百科：

1. minor：在触发缺页异常时，物理页还加载在内存中，仅仅是没有被MMU标记为加载在内存中。
2. major：这就是操作系统按需加载内存的主要机制，发生缺页异常的时候，物理页没有加载到内存中。此时，OS需要找到一个空闲的页加载到内存中，如果不存在空闲的页，需要进行页面置换。
3. invalid：非法地址访问（访问一个不在虚拟地址空间的地址）会触发此类缺页异常，此时用户态程序会coredump。
