---
layout: post
title:  "QNX devb related API"
subtitle: "A recordning from a devb-ufs driver learning"
header-img: img/post-bg-coffee.jpeg
date:   2024-05-20 14:50:00
author: half cup coffee
catalog: true
tags:
    - QNX
    - Resource manager
    - UFS
---

# Background
Not all the description of QNX API you can found from QNX offical online manual. Some APIs only known by QNX and their paratners, such as chip vendor.

Here Lists some APIs description about devb device driver.

It is easy to find an [example](https://github.com/markotikvic/QNX-BBB/tree/master/src/hardware/devb/mmcsd) of devb device driver. Although this driver is 9 years ago. But it is a good reference to know how a device driver connect to QNX CAM layer and block layer. This concpet and most of APIs are not changed in current qnx 7.0 and 7.1.

# devb related APIs

## Function simq_post_ccb( SIMQ *, CCB_SCSIIO * );

It is part of the QNX Neutrino Realtime Operating System's SCSI Interface Module (SIM) layer. This function is used to post a Command Control Block (CCB) to the SIM queue.
*SIMQ* is a pointer to the SIM queue, and *CCB_SCSIIO* is a pointer to the CCB to be posted.
This function does not return a value. If an error occurs while posting the CCB, it's typically indicated by setting a field in the CCB itself.
This function is part of the low-level SCSI interface in QNX, and it's typically used by device drivers or other system-level code. It's not typically used by application-level code.

### SIM

SIM stands for SCSI Interface Module. In the context of the QNX Neutrino Realtime Operating System, the SIM layer provides an interface between the SCSI (Small Computer System Interface) hardware and the software in the system. It's responsible for managing SCSI devices and executing SCSI commands.

### CCB
A SCSI (Small Computer System Interface) I/O command is encapsulated in a Command Control Block (CCB). The CCB for a SCSI I/O command, often represented as CCB_SCSIIO, typically contains the following fields:

```
typedef struct {
    CCB_HEADER cam_ch;  // Common header
    U8 *cam_data_ptr;   // Pointer to data buffer
    U32 cam_dxfer_len;  // Data transfer length
    U8 *cam_cdb_io_ptr; // Pointer to CDB (Command Descriptor Block)
    U8 cam_cdb_len;     // Length of CDB
    U8 cam_sense_len;   // Length of allocated sense
    U8 cam_scsi_status; // SCSI status byte
    U8 cam_scsi_id;     // Target SCSI ID
    U8 cam_scsi_lun;    // Target LUN (Logical Unit Number)
    // Other fields...
} CCB_SCSIIO;
```

The cam_cdb_io_ptr field points to a Command Descriptor Block (CDB), which contains the actual SCSI command to be executed. The structure of the CDB can vary depending on the specific SCSI command.

For example, a CDB for a READ (10) command might look like this:

```
U8 cdb[10] = {
    0x28,  // Operation code for READ (10)
    0,     // Flags
    0, 0,  // Start of Logical Block Address (LBA)
    0, 1,  // End of LBA
    0,     // Reserved
    1, 0,  // Transfer length (number of blocks to read)
    0      // Control
};
```
This is a simplified example. The actual structure of a CCB and a CDB can vary depending on the specific SCSI standard and the specific command.

### cam_configure()
In the QNX operating system, cam_configure is a function that is typically used in the SCSI subsystem. It's part of the Common Access Method (CAM) layer, which provides a common interface for communication between the SCSI Application Layer (SAL) and the SCSI Interface Modules (SIMs).

The cam_configure function is generally used to configure the CAM layer. This could involve tasks such as setting up data structures, initializing hardware, registering SIMs, or other configuration tasks.

### xpt_bus_register()
In the QNX operating system, xpt_bus_register is a function used in the SCSI subsystem, specifically in the Cross Platform Transport (XPT) layer.

The xpt_bus_register function is used to register a SCSI Interface Module (SIM) with the XPT layer. When a SIM is registered, it becomes available for use by the SCSI subsystem to communicate with SCSI devices.

The function typically takes two arguments:

1. A pointer to a sim_hba structure, which represents the SIM that is being registered.

2. A pointer to a cam_sim_softc structure, which represents the software context for the SIM.

Once a SIM is registered with xpt_bus_register, the XPT layer can route I/O requests to it based on the target device of the request.

### simq_ccb_enqueue()
The simq_ccb_enqueue function is typically used to enqueue a CCB (Command Control Block) onto a SIM queue. A CCB is a data structure that represents a SCSI command to be executed by a SCSI device.

The SIM queue is a queue of CCBs that are waiting to be sent to a SCSI device. When a CCB is enqueued, it is added to the end of the SIM queue. The SIM then processes the CCBs in the queue in order, sending each one to the SCSI device and waiting for the device to complete the command.

### simq_ccb_dequeue()
The simq_ccb_dequeue function is typically used to dequeue a CCB (Command Control Block) from a SIM queue. A CCB is a data structure that represents a SCSI command to be executed by a SCSI device.

The SIM queue is a queue of CCBs that are waiting to be sent to a SCSI device. When a CCB is dequeued, it is removed from the front of the SIM queue. The SIM then processes the CCB by sending it to the SCSI device and waiting for the device to complete the command.

### CAM_SIM_ENTRY
In the QNX operating system, CAM_SIM_ENTRY is a macro used in the SCSI subsystem, specifically in the SCSI Interface Module (SIM). The SIM is responsible for communicating with the SCSI controller hardware.

The CAM_SIM_ENTRY macro is used to mark the entry points for the SIM. These functions are called by the Common Access Method (CAM) layer of the SCSI subsystem to perform operations like sending commands to a SCSI device or handling responses from a SCSI device.

### cam_set_thread_state
The cam_set_thread_state function is typically used to set the state of a CAM thread. The CAM layer uses threads to handle various tasks, such as processing I/O requests or handling responses from SCSI devices.

The state of a thread can indicate what the thread is currently doing or what it should do next. For example, a thread might have states like "idle", "busy", or "waiting for response".

### HBA
In the QNX operating system, *SIM_HBA* is a structure used in the SCSI subsystem, specifically in the SCSI Interface Module (SIM).

The *SIM_HBA* structure represents a Host Bus Adapter (HBA). In the context of SCSI, an HBA is a hardware device that provides an interface between the SCSI bus and the host system's bus. The HBA is responsible for sending and receiving data between the SCSI devices and the host system.

The *SIM_HBA* structure typically contains information about the HBA, such as its capabilities, configuration, and state. It also contains pointers to functions that the SIM uses to interact with the HBA.

### sim_alloc_hba()
In the QNX operating system, *sim_alloc_hba* is a function used in the SCSI subsystem, specifically in the SCSI Interface Module (SIM).

The *sim_alloc_hba* function is typically used to allocate memory for a *SIM_HBA* structure. The *SIM_HBA* structure represents a Host Bus Adapter (HBA), which is a hardware device that provides an interface between the SCSI bus and the host system's bus.

The function typically takes as arguments the size of the *SIM_HBA* structure and possibly other parameters related to the configuration of the HBA.

### cam_mphys()
In the QNX operating system, *cam_mphys* is a function used in the SCSI subsystem, specifically in the Common Access Method (CAM) layer.

The *cam_mphys* function is typically used to convert a virtual memory address to a physical memory address. This is often necessary when dealing with hardware devices, such as SCSI devices, that require physical addresses for Direct Memory Access (DMA) operations.

### simq_init()
In the QNX operating system, *simq_init* is a function used in the SCSI subsystem, specifically in the SCSI Interface Module (SIM).

The *simq_init* function is typically used to initialize a SIM queue. A SIM queue is a queue of Command Control Blocks (CCBs) that are waiting to be sent to a SCSI device.

The initialization process typically involves setting up the data structures for the queue and setting the initial state of the queue. This might include setting the queue to be empty and setting any necessary configuration options for the queue.


## UFS driver modeling
### UFS utp transfer request slots
"UTP Transfer Request Slots" is a term used in the context of Universal Flash Storage (UFS) systems. 

UFS is a high-performance interface designed for use in applications where power consumption needs to be minimized, including mobile systems such as smartphones and digital cameras. UFS uses the SCSI Architecture Model and the SCSI command set.

In UFS, UTP (UFS Transport Protocol) is the protocol used for transferring data between the host system and the UFS device. 

A "UTP Transfer Request Slot" is a slot in the UFSHCI (UFS Host Controller Interface) where a UTP Transfer Request can be placed. Each slot corresponds to a command that the host wants the UFS device to execute. 

The host writes the command (and any associated data) into a UTP Transfer Request Descriptor, and then signals the UFS device to process the command by writing to the UTP Transfer Request Run Stop Register (UTRLRSR) and the UTP Transfer Request Doorbell Register (UTRLDBR).

### UTP command
In the context of Universal Flash Storage (UFS), UTP (UFS Transport Protocol) commands are the commands that are sent from the host system to the UFS device to perform various operations.

UFS is a high-performance interface designed for use in applications where power consumption is a critical factor, such as mobile systems. UFS uses the SCSI command set for data transport, but it has its own transport protocol, UTP.

A UTP command is encapsulated in a UTP Command Descriptor (UCD). The UCD contains all the information needed to process the command, including the operation code (opcode), the logical unit number (LUN), the command argument, and the data direction (read or write).

There are various types of UTP commands, including:

* Read: Read data from the UFS device.
* Write: Write data to the UFS device.
* Inquiry: Request information about the UFS device.
* Test Unit Ready: Check if the UFS device is ready to transfer data.
* Mode Sense: Request mode parameters from the UFS device.
* Mode Select: Send mode parameters to the UFS device.

### UTP Transfer Request Descriptor
UTP Transfer Request Descriptor (UTRD) is a data structure that represents a UFS Transport Protocol (UTP) command to be executed by a UFS device.

The UTRD contains all the information needed to process the UTP command, including:

* The operation code (opcode) of the UTP command.
* The logical unit number (LUN) that the command is targeted at.
* The command argument, which provides additional information for the command.
* The data direction (read or write).
* The data buffer, which contains the data to be written or the space for the data to be read.

The UTRD is typically stored in memory that is accessible by both the host system and the UFS device, and the physical address of the UTRD is passed to the UFS device to initiate the command.


### UTP Task Management
UTP Task Management refers to a set of commands that are used to control and manage the execution of UTP commands.

These task management commands are encapsulated in UTP Task Management Request Descriptors (UTMRDs). The UTMRD contains all the information needed to process the task management command, including the task management function to be performed and the logical unit number (LUN) that the function is targeted at.

There are various types of UTP task management functions, including:

* Abort Task: Aborts a specific command.
* Abort Task Set: Aborts all commands for a specific LUN.
* Clear Task Set: Clears all commands for a specific LUN.
* Logical Unit Reset: Resets a specific LUN.
* Target Reset: Resets the UFS device.

### SLOTS
A slot refers to a position in the command queue where a command can be placed for execution. This is similar to the concept of "IO depth" in other storage systems.

Each slot can hold a UTP command, which is encapsulated in a UTP Transfer Request Descriptor (UTRD). The UFS device can process multiple commands concurrently, up to a limit defined by the device. This limit is often referred to as the number of "slots" or the "queue depth".

When a command is issued, it is placed in a free slot in the command queue. The UFS device then processes the commands in the queue, possibly in parallel. When a command is completed, its slot becomes free and can be used for another command.
