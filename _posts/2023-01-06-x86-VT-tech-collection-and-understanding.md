---
layout: post
title:  "x86 VT tech collection and understanding"
date:   2023-01-06 17:43:59
author: half cup coffee
categories: Virtualization
tags:	Virtualization
---

# A summary introduction of x86\-64

### Why Virtualization?

CPU 的性能相比从前有很大的提升\,具备支持多个操作系统的资源

节约服务器部署空间\(节约车内空间\)

节约服务器部署成本\(节约成本\,同样适用与车\)

根据实际情况动态的性能迁移\(dynamic resource management\)

### Virtualization type

#### Type 1

Small shim layer runs on hardware

Create operating environment which guest OS can run\. Resource assignment
![](/assets/x86-VT-tech/img/10.png)

#### Type 2

A complete OS which with the ability to host another complete OS within a host process

Acceptable for desktop


![](/assets/x86-VT-tech/img/11.png)

## current solution on the market

![](/assets/x86-VT-tech/img/12.png)

## What a operation system depends on?

VM OS software is same as the OS direct running on HW\, No difference\.

![](/assets/x86-VT-tech/img/13.png)

## VM OS doesn’t know it is running in VM environment

* CPU virtualization

* Interrupt virtualization

* Memory virtualization

* IO device virtualization

* Graphic virtualization

## Intel X86 Hardware acceleration for virtualization

![](/assets/x86-VT-tech/img/14.png)

![](/assets/x86-VT-tech/img/15.png)

![](/assets/x86-VT-tech/img/16.png)

![](/assets/x86-VT-tech/img/17.png)

## Intel's technology for virtualization on the x86 platform

virtual\-machine extensions\(VMX\)

Define processor\-level support for virtual machines on IA processors

VMX instructions are provided

Adds ten new instructions: VMPTRLD\, VMPTRST\, VMCLEAR\, VMREAD\, VMWRITE\, VMCALL\, VMLAUNCH\, VMRESUME\, VMXOFF\, VMXON

Import new processor operation:  VMX operation

VMX root operation

VMX non\-root operation

Two principal classes of software

Virtual\-machine monitors \(VMM\)

## IA privilege level

![](/assets/x86-VT-tech/img/18.png)

User space applications

## VMX root operation

![](/assets/x86-VT-tech/img/19.png)

![](/assets/x86-VT-tech/img/110.png)

## Lifecycle of VMM

![](/assets/x86-VT-tech/img/111.png)

![](/assets/x86-VT-tech/img/112.png)

__Virtual machine control structure__

![](/assets/x86-VT-tech/img/113.png)

__Guest\-state area__

__VM\-exit control fields__

__VM\-entry control fields__

__VM\-execution control fields__

__VM\-exit information fields__

## APIC Virtualization and Virtual Interrupts

APIC\(Advanced Programmable interrupt controller \)

![](/assets/x86-VT-tech/img/114.png)

PIC \(Programmable interrupt controller \)

### Split architecture design



#### Local APIC \(LAPIC\) for every processor

* manage all external interrupts for some specific processor

* accept and generate inter\-processor interrupts \(IPIs\) between LAPICs

#### I/O APIC on a system bus

* route the interrupts it receives from peripheral buses to one or more local APICs

![](/assets/x86-VT-tech/img/115.gif)

## APIC Virtualization and Virtual Interrupts

Accesses to the APIC\, track the state of the virtual APIC\, and deliver virtual interrupts — all in VMX non\-root operation with out a VM exit

![](/assets/x86-VT-tech/img/116.png)

#### APIC\-register virtualization

Redirects most guest APIC reads/writes to virtual\-APIC page

virtual\-APIC page : a 4K page memory used to access the APIC

Most reads will be allowed without VM exits

VM exits occur after __ __ writes \(no need for decode\)

#### Virtual\-interrupt delivery

Extend TPR\(Task priority\) virtualization to other APIC registers

No need for VM exits for most frequent accesses

CPU delivers virtual interrupts to guest \(including virtual IPIs\)

process posted interrupts

## extended page\-table mechanism \( __EPT__ \)


* Address Translation
  * Guest OS expects contiguous\, zero\-based physical memory
  * VMM must preserve this illusion
* Page\-table Shadowing
  * VMM intercepts paging operations
  * Constructs copy of page tables
* Overheads
  * VM exits add to execution time
  * Shadow page tables consume significant host memory

![](/assets/x86-VT-tech/img/117.gif)


![](/assets/x86-VT-tech/img/118.gif)

With hardware support for nested paging caches both the Virtual memory \(Guest OS\) to Physical memory \(Guest OS\) as the Physical Memory \(Guest OS\) to real physical memory transition in the TLB\. The TLB has a new  __VM specific tag__ \, called the Address Space IDentifier \(ASID\)\. This allows the  __TLB to keep track of which TLB entry belongs to which VM__ \. The result is that a VM switch does not flush the TLB\. The TLB entries of the different virtual machines all coexist peacefully in the TLB

* Extended Page Tables \(EPT\)
  * Map guest physical to host address
  * New hardware page\-table walker
  * A guest OS can modify its own page tables freely and without VM exits
  * A single EPT supports entire VM:  instead of a shadow pagetable per guest process

## VIRTUAL PROCESSOR IDENTIFIERS \(VPIDS\)

The original architecture for VMX operation required VMX transitions to flush the TLBs and paging\-structure caches\.

This ensured that translations cached for the old linear\-address space would not be used after the transition\.

__VPIDs__  introduce to VMX operation a facility by which a logical processor may cache

information for multiple linear\-address spaces\. When VPIDs are used\, VMX transitions may retain cached information and the logical processor switches to a different linear\-address space\.

## VT\-d

### Intel Virtualization Technology for Directed I/O

### A VMM may support various models for I/O virtualization:

* Emulating the device API

* assigning physical I/O devices to VMs\(\)

* permitting I/O device sharing in various manners

### Key capabilities

* allows to assign I/O devices to VMs in any desired configuration

* DMA remapping\. Supports address translations for device DMA data transfers

* Interrupt remapping\. Provides VM routing and isolation of device interrupts

* Reliability features\. Reports and records system software DMA and interrupt erros that may otherwise corrupt memory of impact VM isolation\.

## Understanding DMAR\(DMA remapping\)

When a VM or a Guest is launched over the VMM\, the address space that the Guest OS is provided as its physical address range\, known as  __Guest Physical Address \(GPA\)__ \, may not be the same as the real  __Host Physical Address \(HPA\)__ \. DMA capable devices need HPA to transfer the data to and from physical memory locations\. However\, in a direct assignment model\, the guest OS device driver is in control of the device and is providing GPA instead of HPA required by the DMA capable device\. DMA remapping hardware can be used to do the appropriate conversion\. Since the GPA is provided by the VMM it knows the conversion from the GPA to the HPA\. The VMM programs the DMA remapping hardware with the GPA to HPA conversion information so the DMA remapping hardware can perform the necessary translation\. Using the remapping\, the data can now be transferred directly to the appropriate buffer of the guests rather than going through an intermediate software emulation layer\.

![](/assets/x86-VT-tech/img/119.png)

TRANSLATION:

具有DMA能力的物理设备往往使用HPA的物理内存\, 在direct assign modle中\, 是VM中的设备驱动程序在控制和提供GPA而不是HPA给这个设备\.DMAR这时就被用来做GPA到HPA的地址转换\.

在VMM中设置DMAR\, 其中包含 GPA 和HPA 的信息\, 这样DMAR的硬件就可以做它们之间的转换\. 这样\,具有DMA能力的物理设备就可以直接和 VM的内存交换数据了

## Graphics Virtualization

__GVT-d__ : 

Dedicatedly assigned to a virtual machine

__GVT-s__ : 

VM using a virtual graphics driver

Shared between multiple virtual machines by

__GVT-g__:

VM using native graphics driver

Shared between multiple virtual machines under time\-sharing

Hypervisor directly assigns the full GPU resource to each virtual machine\. Thus\, during its time slice\, while the virtual machine gets a full dedicated GPU\, from overall system view point several virtual machines share a single GPU\.

## Graphics Virtualization

vGPU device model

![](/assets/x86-VT-tech/img/120.png)

![](/assets/x86-VT-tech/img/121.png)

# Virtualization in the ARM Cortex™ Processors

__How does ARM do Virtualization__

* Second stage of address translation \(separate page tables\)

* Functionality for virtualizing interrupts inside the Interrupt Controller

* Functionality for virtualizing all CPU features\, including CP15

* Option of a MMU within the system to help virtualize IO

* New Non\-secure level of privilege to hold Hypervisor

![](/assets/x86-VT-tech/img/122.png)

__Privilege__

* Guest OS same kernel/user privilege structure

* HYP mode higher privilege than OS kernel level

* VMM controls wide range of OS accesses

* Hardware maintains TZ security \(4th privilege\)

## Virtualization in the ARM Cortex™ Processors

__Virtual Memory in Two StageS__

![](/assets/x86-VT-tech/img/123.png)


__Virtual interrupt(Virtual GIC )__

* Physical registers and virtual registers

* Non\-virtualized system and hypervisor access the physical registers

* Virtual machines access the virtual registers

* Guest OS functionality does not change when accessing the vGIC

* Virtual registers are remapped by hypervisor

* Interrupts are configured to generate a Hypervisor trap

Hypervisor can deliver an interrupt to a CPU running a virtual process

![](/assets/x86-VT-tech/124.png)

