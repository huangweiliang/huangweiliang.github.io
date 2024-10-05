---
layout: post
title:  "MPHY short introduction"
subtitle: "An interface be used for UFS storage"
header-img: img/post-bg-coffee.jpeg
date:   2024-10-04 08:00:00
author: half cup coffee
catalog: true
tags:	
    - Linux
    - Android
---

# MPHY

The MIPI M-PHY is a serial communication protocol that aims to reduce power consumption and increase data rates for mobile systems: 

## Power efficiency
MIPI M-PHY uses burst operation to switch between power saving and data transmission modes, which reduces power consumption. 

## High data rates
MIPI M-PHY is designed for high-bandwidth applications and can support ultra-high bandwidths up to 11.6 Gb/s. 

## Low pin counts
MIPI M-PHY is designed to obtain low pin counts. 

## Lane scalability
MIPI M-PHY offers lane scalability to meet different bandwidth requirements. 

MIPI M-PHY is used in a variety of applications, including: Connecting flash memory-based storage, Connecting cameras and RF subsystems, Chip-to-chip inter-processor communications (IPC), and Supporting USB 3.0 protocol in SSIC. 
MIPI M-PHY is the foundation for several upper layer protocols, including DigRF, UniPro, LLI, and CSI-3.

# MPHY Line status
There are 4 kinds of Line state of MPHY. 
* DIF-P: A positive differential voltage driven by TX
* DIF-N: A negative differential voltage driven by TX
* DIF-Z: A zero differential voltage maintained by RX
* DIF-Q: An unknow floating voltage, or no line drive

# MPHY state machine

There are 2 operating modes: HS-MODE and LS-MODE.
For each operating mode there are save state. STALL is the save state of HS-MODE. SLEEP is the save state of LS-MODE.

HIBERN8 state is the ultra low power consumption mode which only maintaining the configuration settings. 

The detail of the state machine diagram is showing like below ：
Please note that for every state switching it is together with the line state change. So line state is important signal for the state indication. For example, the state switchs from PWM-BURST to the SLEEP. The line state need change from the DIF-P to DIF-N.  The SLEEP state is confirmed after the state is change completely to the DIF-N.

![Crepe](/img/The-M-PHY-state-machine.png)

