---
layout: post
title: "intel_mid: Overview of Simple Platform Interface"
date: 2013-11-27 15:40
comments: true
categories: [android, kernel, firmware, intel]
---
Start to develop android kernel under X86 MID platform, merrifield.
I am a newbie for porting kernel to Atom-based platform but am familiar with linux kernel and device drivers.
Noticed that Intel Android BSP introudes Simple Firmware Interface (SFI) a method for platform firmware to export
static tables (I2C, SPI, GPIO) to the operation system.

Intel's newer Atom processors support SFI since "Moorestown" SoC and SFI implementation was merged into upstream kernel 2.6.32(http://lwn.net/Articles/340476)

Actually, below link descrbies the SFI and explains how does SFI related to ACPI and UEFI.
https://simplefirmware.org/faq

Besides, below patch sets are to refactor existing code and implement a flexible way to support multiple boards and devices.
https://lkml.org/lkml/2013/10/10/81

/arch/x86/platform/intel-mid/intel_mid_sfi.c is SFI parsing implementation, and let me understand how get_gpio_by_name() works.

