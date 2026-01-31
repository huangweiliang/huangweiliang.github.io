---
layout: post
title:  "Understanding QNX Hypervisor Shared Memory"
subtitle: "A Deep Dive into Inter-Guest Communication"
header-img: img/post-bg-coffee.jpeg
date:   2026-01-31 09:00:00
author: half cup coffee
comments: true
catalog: true
tags:	
    - QNX
---


# Understanding QNX Hypervisor Shared Memory

*Published: January 2026*

## Introduction

In modern automotive and embedded systems, hypervisors enable multiple operating systems to run concurrently on a single hardware platform. The QNX Hypervisor provides a robust virtualization solution that allows safety-critical QNX systems to coexist with feature-rich Android/Linux guests. One of the key challenges in such heterogeneous environments is efficient inter-guest communication. This article explores the QNX Hypervisor's shared memory mechanism, diving deep into its architecture, implementation challenges, and a critical resource leak issue we discovered during production deployment.

## The Problem: Inter-Guest Communication in Virtualized Systems

When running multiple operating systems under a hypervisor, traditional IPC mechanisms don't work across guest boundaries. You can't use:
- **Sockets** - Each guest has isolated network stacks
- **Pipes/FIFOs** - File systems are isolated
- **Message queues** - Kernel-specific, don't cross VM boundaries
- **Shared memory (traditional)** - Each guest has its own address space

The QNX Hypervisor solves this with a **hardware-assisted shared memory device** that provides:

1. **Zero-copy data transfer** between guests
2. **Interrupt-based notifications** for synchronization  
3. **Connection management** for up to 16 concurrent clients
4. **Memory safety** through hypervisor mediation

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    QNX Hypervisor                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐           ┌──────────────────┐        │
│  │   QNX Host      │           │  Linux/Android   │        │
│  │   Guest         │           │  Guest           │        │
│  │                 │           │                  │        │
│  │  Shmem Driver   │◄─────────►│  xx-shmem.ko    │        │
│  │  (QNX native)   │  Notify   │  (Linux driver) │        │
│  └────────┬────────┘           └────────┬─────────┘        │
│           │                              │                  │
│           └──────────┬───────────────────┘                  │
│                      │                                      │
│         ┌────────────▼─────────────────────┐               │
│         │   Virtual Shared Memory Device   │               │
│         │                                   │               │
│         │  ┌─────────────────────────────┐ │               │
│         │  │  Factory Page (MMIO)        │ │               │
│         │  │  - signature                │ │               │
│         │  │  - shmem (paddr)            │ │               │
│         │  │  - vector (IRQ)             │ │               │
│         │  │  - status                   │ │               │
│         │  │  - size (triggers creation) │ │               │
│         │  │  - name[32]                 │ │               │
│         │  └─────────────────────────────┘ │               │
│         │                                   │               │
│         │  ┌─────────────────────────────┐ │               │
│         │  │  Shared Memory Region       │ │               │
│         │  │                             │ │               │
│         │  │  Page 0: Control Page       │ │               │
│         │  │    - status (R/O)           │ │               │
│         │  │    - idx (connection ID)    │ │               │
│         │  │    - notify (W/O)           │ │               │
│         │  │    - detach (W/O)           │ │               │
│         │  │                             │ │               │
│         │  │  Page 1-N: Data Pages       │ │               │
│         │  │    (Application data)       │ │               │
│         │  └─────────────────────────────┘ │               │
│         └───────────────────────────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## How It Works: The Creation Protocol

### Step 1: Device Discovery

The shared memory device can be discovered via:

**PCI-attached mode:**
```c
// Device ID: 0x1C05:0x0001 (QNX vendor ID)
static const struct pci_device_id shmem_pci_id[] = {
    { PCI_DEVICE(0x1C05, 0x0001), 0 },
    { 0 }
};
```

**Memory-mapped mode (Device Tree):**
```dts
shmem@1b018000 {
    compatible = "qvm,guest_shm";
    reg = <0x0 0x1b018000 0x0 0x1000>;
    interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
};
```

### Step 2: Creating a Shared Memory Region

The creation process uses the `guest_shm_create()` inline function:

```c
static inline void
guest_shm_create(volatile struct guest_shm_factory *factory, unsigned size) {
    // Memory barriers prevent compiler reordering
    asm volatile( "" ::: "memory");
    factory->size = size;  // Writing to size triggers hypervisor action
    asm volatile( "" ::: "memory");
}
```

**Why memory barriers?** The name must be written to the factory page *before* triggering creation. The barriers ensure:
1. `factory->name` write completes first
2. `factory->size` write happens atomically
3. No subsequent reads get reordered before creation completes

**What happens in the hypervisor:**

```
Guest writes factory->size
       ↓
Hypervisor traps MMIO write
       ↓
Hypervisor checks factory->name
       ↓
Region exists? ──Yes──→ Attach to existing region
       │
       No
       ↓
Allocate physical memory
       ↓
Create connection entry (index 0-15)
       ↓
**Allocate sigevent for notifications** ← CRITICAL
       ↓
Return control page paddr in factory->shmem
```

### Step 3: Mapping the Control Page

After creation succeeds:

```c
uint64_t paddr = factory->shmem;
data->ctrl = memremap(paddr, 
                      (factory->size + 1) * PAGE_SIZE, 
                      MEMREMAP_WB);

// Control page structure
struct guest_shm_control {
    uint32_t status;   // [31:16]=active clients, [15:0]=notify bits
    uint32_t idx;      // This guest's connection index (0-15)
    uint32_t notify;   // Write bitset to notify other guests
    uint32_t detach;   // Write to disconnect
};
```

The **connection index** (`ctrl->idx`) is crucial - it identifies this guest's connection and determines which bit represents it in status bitsets.

### Step 4: Notification Protocol

**Sending a notification:**
```c
// Notify all other guests
data->ctrl->notify = ~0;  // Special value: "everyone but me"

// Or notify specific guests
data->ctrl->notify = (1 << 2) | (1 << 5);  // Notify guests 2 and 5
```

**Receiving notifications (interrupt-driven):**
```c
static irqreturn_t shmem_irq(int irq, void *arg) {
    // status[15:0] = who notified us (bitset)
    // status[31:16] = currently active connections
    unsigned status = data->ctrl->status;
    unsigned notifiers = status & 0xFFFF;
    unsigned active_clients = status >> 16;
    
    // Process notifications...
    return IRQ_HANDLED;
}
```

## The Resource Leak: A Case Study

### The Symptom

During production testing, we observed:
```
[  0.000] System boot OK
[  120.456] QVM running normally
[  185.234] QVM ERROR: RLIMIT_SIGEVENT_NP exhausted
[  185.235] System crash - hypervisor terminated
```

After ~32 attach/detach cycles, the QNX hypervisor would crash with "RLIMIT_SIGEVENT_NP exceeded."

### What is RLIMIT_SIGEVENT_NP?

`RLIMIT_SIGEVENT_NP` is a QNX-specific resource limit controlling the maximum number of **sigevent** structures a process can allocate. Sigevents are used for:
- Timers (`timer_create()`)
- Message pulses (`MsgDeliverEvent()`)
- **Interrupt delivery to virtualized guests** ← Our issue
- Asynchronous I/O notifications

Default limit: **32-64 sigevents** per process

### Root Cause Analysis

The Linux driver had a critical bug in its attach/detach logic:

**Buggy code:**
```c
static int shmem_attach(struct file *filp, size_t pageCount) {
    // ❌ NO CHECK if already attached!
    
    guest_shm_create(factory, pageCount);
    
    // ❌ OVERWRITES previous ctrl pointer if called twice
    data->ctrl = memremap(paddr, size, MEMREMAP_WB);
    
    return 0;
}

static int shmem_detach(struct file *filp) {
    if (data->ctrl) {
        data->ctrl->detach = 0;
        memunmap(data->ctrl);
        data->ctrl = NULL;
    }
    // ❌ Hypervisor connection cleaned up
    // ❌ But no corresponding free of initial resources!
    return 0;
}
```

**The leak scenario:**

```c
// User space misbehavior (intentional or bug):
fd = open("/dev/xx-shmem", O_RDWR);

ioctl(fd, BUFIOC_ATTACH, &pages);  // Creates connection #3, allocates sigevent
// ... forgot to detach ...
ioctl(fd, BUFIOC_ATTACH, &pages);  // Creates connection #4, allocates sigevent
                                    // Connection #3 is now ORPHANED!

ioctl(fd, BUFIOC_DETACH);          // Only cleans up connection #4
close(fd);

// Result: Connection #3's sigevent is LEAKED in hypervisor
```

**After 32 iterations:**

```
Hypervisor sigevent pool:
[sev0] [sev1] ... [sev31] ALL LEAKED!
         ▲      ▲       ▲
         └──────┴───────┘
    All allocated to orphaned connections

Iteration 33:
❌ CRASH: Cannot allocate sigevent for new connection
```

### The Fix

Add state checking to prevent multiple attaches:

```c
static int shmem_attach(struct file *filp, size_t pageCount) {
    struct shmem_data *data = get_driver_data(filp);
    
    // ✅ CHECK: Already attached?
    if (data->ctrl != NULL) {
        pr_err("Already attached! Must detach first.\n");
        return -EBUSY;  // Return error to user space
    }
    
    guest_shm_create(factory, pageCount);
    
    if (factory->status != GSS_OK) {
        return -factory->status;
    }
    
    data->ctrl = memremap(factory->shmem, size, MEMREMAP_WB);
    if (!data->ctrl) {
        // ⚠️ memremap failed but hypervisor allocated connection
        // This is a corner case that needs handling
        return -ENOMEM;
    }
    
    // Success
    return 0;
}
```

**Improved detach with validation:**

```c
static int shmem_detach(struct file *filp) {
    struct shmem_data *data = get_driver_data(filp);
    
    if (!data->ctrl) {
        pr_warn("Detach called but not attached\n");
        return 0;  // Idempotent
    }
    
    pr_info("Detaching from connection idx %u\n", data->ctrl->idx);
    
    // Tell hypervisor to release this connection
    data->ctrl->detach = 0;
    
    // Unmap kernel memory
    memunmap(data->ctrl);
    
    // Clear state
    data->ctrl = NULL;
    data->shmdata = NULL;
    data->pagesize = 0;
    
    return 0;
}
```

## Memory Layout and Data Sharing

### Typical Configuration

For a 64KB shared memory region (16 pages):

```
Physical Address: 0x8000_0000

Offset 0x0000:  ┌──────────────────────────┐
                │  Control Page (4KB)      │
                │  - status, idx, notify   │
                └──────────────────────────┘
Offset 0x1000:  ┌──────────────────────────┐
                │  Guest 0 Buffer (4KB)    │
                └──────────────────────────┘
Offset 0x2000:  ┌──────────────────────────┐
                │  Guest 1 Buffer (4KB)    │
                └──────────────────────────┘
                │         ...              │
Offset 0xF000:  ┌──────────────────────────┐
                │  Guest 15 Buffer (4KB)   │
                └──────────────────────────┘
```

### Application Protocol Example

**Producer (QNX guest):**
```c
// Write data to our buffer
struct my_data *buf = (struct my_data *)(shmem + (idx * 4096));
buf->frame_number = 1234;
buf->timestamp = get_time();
memcpy(buf->payload, frame_data, frame_size);

// Notify Linux consumer (guest idx=2)
ctrl->notify = (1 << 2);
```

**Consumer (Linux guest):**
```c
// IRQ handler wakes us
wait_event_interruptible(wait_queue, data_ready);

// Read data from producer's buffer
struct my_data *buf = (struct my_data *)(shmem + (producer_idx * 4096));
process_frame(buf->payload, buf->timestamp);
```

## Performance Considerations

### Advantages

1. **Zero-copy**: Direct memory access, no hypervisor involvement for data transfer
2. **Low latency**: ~1-5 microseconds for notification delivery
3. **High bandwidth**: Limited only by memory bandwidth (~10 GB/s typical)

### Limitations

1. **Manual synchronization**: Application must implement locking/atomics
2. **Fixed client limit**: Maximum 16 concurrent guests
3. **Memory overhead**: One control page per region
4. **Notification granularity**: Per-guest, not per-message

### Best Practices

**DO:**
- ✅ Use separate regions for different purposes (control vs. data)
- ✅ Implement application-level flow control
- ✅ Use atomic operations for producer-consumer synchronization
- ✅ Validate data->ctrl != NULL before access
- ✅ Call detach before close

**DON'T:**
- ❌ Call attach multiple times without detach
- ❌ Assume notifications are reliable (they're sticky bits)
- ❌ Access shared memory without proper synchronization
- ❌ Leak attachments (always pair attach/detach)
- ❌ Assume specific connection indices (use ctrl->idx)

## Testing and Validation

We developed test tools to validate the driver:

```c
// Test repeated attach/detach cycles
./test_attach_detach /dev/xx-shmem 50 100

// Expected with bug: Fails around iteration 32
// Expected with fix: All 50 iterations succeed
```

The test clearly demonstrates resource exhaustion:
```
[0032/0050] ATTACH OK -> DETACH OK
[0033/0050] ATTACH FAILED: Cannot allocate memory (errno=12)
  ⚠ RESOURCE EXHAUSTION! This indicates the leak!
```

## Debugging Tips

### Kernel Debugging

**Enable driver debug messages:**
```bash
echo 8 > /proc/sys/kernel/printk  # Enable debug level
dmesg -w | grep xx-shmem
```

**Monitor hypervisor resources:**
```bash
# On QNX host
pidin | grep qvm
sloginfo -w | grep sigevent
```

### Common Issues

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| "Cannot allocate memory" after ~32 calls | sigevent leak | Apply state checking fix |
| "Device or resource busy" | Already attached | Add ctrl == NULL check |
| Kernel NULL pointer deref | Accessing ctrl before attach | Validate ctrl in all paths |
| Notifications not received | Wrong IRQ or not registered | Check platform_get_irq() |
| Data corruption | Race condition | Add proper locking |

## Conclusion

The QNX Hypervisor's shared memory mechanism provides an elegant solution for inter-guest communication in mixed-criticality systems. However, as we discovered, improper resource management at the driver level can lead to subtle but critical leaks that only manifest under sustained operation.

Key takeaways:

1. **State validation is critical**: Always check if resources are already allocated
2. **Resource lifecycle matters**: Every allocation must have a corresponding deallocation
3. **Hypervisor resources are limited**: sigevent pools are finite and shared
4. **Testing at scale reveals issues**: Bugs may not appear in basic functional tests
5. **Defense in depth**: Validate at both kernel driver and user space levels

This investigation reinforces the importance of rigorous resource management in virtualized environments, where resource exhaustion in the hypervisor can bring down the entire system—not just a single guest.

## References

- QNX Hypervisor 2.2 Documentation
- QNX Neutrino RTOS System Architecture Guide
- Linux Kernel Device Driver Documentation
- "Virtualization in Safety-Critical Systems" - SAE International


---

*Have questions or found this useful? Feel free to reach out or open an issue on GitHub.*

*Tags: #QNX #Hypervisor #Virtualization #Automotive #EmbeddedLinux #SharedMemory #ResourceManagement*
