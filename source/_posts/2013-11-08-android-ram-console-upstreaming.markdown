---
layout: post
title: "Android ram-console upstreaming"
date: 2013-11-08 17:45
comments: true
categories: [android, kernel]
---
While dealing with board bring-up powered by Tegra5, starts to aware of this upstream change for ram console (aka /proc/last_kmsg). A working group, Android Upstreaming, comes from Linaro foundation to merge ram_console into pstore framework ( http://lwn.net/Articles/497881/).

The Android Upstreaming team's mission is to reduce and eventually eliminate the differences between the upstream kernel and the Android kernel. The team works closely with Google and upstream kernel developers to find ways to implement Android required features in a way that meets the need of both communities.

There are currently two competing debug facilities to store kernel messages in a persistent storage: a generic pstore and Google's persistent_ram by Colin Cross. Not so long ago (https://lkml.org/lkml/2012/3/8/252) noticed by Greg KH@ARM Linux, it was decided to fix this situation. There is a buleprint registered by Linaro Linux to descrbie those debug facilities at https://blueprints.launchpad.net/linux-linaro/+spec/android-ram-console 

To follow up android upstreaming's works, I remove legacy driver supporting for ram console and persistent ram and switch new pstore framework. In other words, start to looking into /sys/fs/pstore/console-ramoops as we used to did analysis on /proc/last_kmsg for dying moment across system reboot.

My works were merged into Asus internal development branch jb-mr2-t50-k3.10, but not opened yet. We could refer to https://android.googlesource.com/kernel/tegra/+/android-tegra-3.10 for further reference. 
