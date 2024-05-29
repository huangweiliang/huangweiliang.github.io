---
layout: post
title:  "Cache and TLB introduction"
subtitle: "Basic deep knowledge for ARM"
header-img: img/post-bg-coffee.jpeg
date:   2024-05-26 08:43:59
author: half cup coffee
catalog: true
tags:
    - Arm64
    - Cache
---

# What is Cache ?  

See this description from [ARM document]:

*A cache is a small, fast block of memory that sits between the core and main memory. It holds copies of items in main memory. Accesses to the cache memory occur significantly faster than those to main memory. Whenever the core reads or writes a particular address, it first looks for it in the cache. If it finds the address in the cache, it uses the data in the cache, rather than performing an access to main memory.*

![Crepe](/img/cache-tlb-1.png)

There are different type of Cache and different level of Cache.  For example,  there are data cache only for data which is called d-cache, and instruction  cache only for instructions which is called I-cache.

In nowadays,  in the multi-core system.  There are different cache level.  For example, like the above picture, Level1 only be used by each core.  Level 2 is shared by a cluster. Level 3 is shared by all clusters.
It is not distinguish data or instruction in cache L2 and L3.  the higher level cache means slower speed.  But compare to main memory, it still very faster.

When CPU core try to access a data.  It will first check if the data is available in these caches.  If it can be found , we call it cache hit.  If the data is not found in these caches we called it cache missing.  When the cache missing happens,  the CPU core will fetch the data from main memory.   This process relate to the topic about virtual address to physical address translation.

A simplified four-way set associative 32KB L1 cache (such as the data cache of the Cortex-A57
processor), with a 16-word (64 byte) cache line length:

![Crepe](/img/cache-tlb-2.png)

# TLB

Here is a good explanation about TLB from :  [TLB intro article]

*TLB: Translation Lookaside Buffer is required only if Virtual Memory is used by a processor.*

*In short, TLB speeds up the translation of virtual addresses to a physical address by storing page-table in faster memory. In fact, TLB also sits between CPU and Main memory. Precisely speaking, TLB is used by MMU when a virtual address needs to be translated to a physical address. By keeping this mapping of virtual-physical addresses in fast memory, access to page-table improves. It should be noted that page-table (which itself is stored in RAM) keeps track of where virtual pages are stored in the physical memory. In that sense, TLB also can be considered as a cache of the page table.*

[ARM document]:

*Each TLB entry typically contains not just physical and Virtual Addresses, but also attributes such as memory type, cache policies, access permissions, the Address Space ID (ASID), and the Virtual Machine ID (VMID).*

*If the TLB does not contain a valid translation for the Virtual Address issued by the processor, known as a TLB miss, an external translation table walk or lookup is performed. Dedicated hardware within the MMU enables it to read the translation tables in memory. The newly loaded translation can then be cached in the TLB for possible reuse if the translation table walk does not result in a page fault.*


##  kernel and application Virtual Address spaces

[ARM document]:
*The table base addresses are specified in the Translation Table Base Registers (TTBR0_EL1) and (TTBR1_EL1). The translation table pointed to by TTBR0 is selected when the upper bits of the VA are all 0. TTBR1 is selected when the upper bits of the VA are all set to 1.*

*kernel space can be mapped to the most significant area of memory and the Virtual Address space associated with each application mapped to the least significant area of memory. However, both of these are mapped to a much smaller Physical Address space.*

![Crepe](/img/cache-tlb-3.png)

## Translating a Virtual Address to a Physical Address

There are several information we shall know:
Physical memory is used by page by page.  The DRAM which we call main memory is spited piece by piece which each piece is 4KB typically. (it is configurable, but typically 4KB).  So there is a concept to index the physical memory is by page.  

![Crepe](/img/cache-tlb-4.png)

These 2 address space has its own page table.  So 3 steps to do the translation:

`Decide the page table base address`

`Find the index in the table`

`Get the physical address`
    
[ARM document]:

*In practice, an OS can further divide a large section of virtual memory into smaller pages. For
a second-level table, the first-level descriptor contains the physical base address of the
second-level page table. The Physical Address that corresponds to the Virtual Address requested
by the processor, is found in the second-level descriptor.*

![Crepe](/img/cache-tlb-5.png)



[ARM document]: https://documentation-service.arm.com/static/5fbd26f271eff94ef49c7020?token=
[TLB intro article]: https://www.geeksforgeeks.org/whats-difference-between-cpu-cache-and-tlb/
[zhihu cache intro]: https://zhuanlan.zhihu.com/p/108425561

