---
layout: post
title: "Android RenderScript Notes"
date: 2013-12-12 12:06
comments: true
categories: [android, renderscript] 
---

This page has my notes from initial study for Android RenderScript.  

RenderScript is Android 3D graphics rendering and intensive computation using heterogeneous computing.  
1. Portability: A general purpose compute API across different system computing hardware  
2. High Performance  
3. Developer Friendly: A compute API similar to CUDA, OpenCL or GLSL, and a familiar language with C99  

There are three major components in RenderScript  
1. Offline compiler (llvm-rs-cc): Convert script files into portable bitcode and reflected Java layer  
2. Online JIT compiler (libbcc): Translate portable bitcode to appropriate machine code (CPU/GPU/DSP/...)  
3. Runtime library support (libRS): Manage scripts from Dalvik layer and also provide basic support libraries (math functions, etc.)  
A good introduciton of [Android RenderScript on LLVM](https://events.linuxfoundation.org/slides/2011/lfcs/lfcs2011_llvm_liao.pdf) is given by Shih-Wei Liao, and a good comparsion and analysis of different [android programming model](https://www.google.com.tw/url?sa=t&rct=j&q=&esrc=s&source=web&cd=6&cad=rja&ved=0CGEQFjAF&url=http%3A%2F%2Fwww.kandroid.org%2Fboard%2Fdata%2Fboard%2Fconference%2Ffile_in_body%2F1%2F6renderscript_2013_10_24_%25EC%2588%2598%25EC%25A0%2595.pdf&ei=ACapUtGyKoGFoQSFv4GgBQ&usg=AFQjCNGj_OC0Yl6ssx7s1EDFii2WmfLyiw&sig2=Ng96XAtyU2Zkc2o7EGbA8g) is given by Kandroid S/W Fundamentals Study Group at 12th Kandroid Conference, 2013. 

Clang is offline frontend compiler to create LLVM bitcode and reflected Java layer. The portable bitcode supplied as a resource within .apk container and is compiled before use one the device. Offline compiler, llvm-rs-cc, performs machine-independent optimizations on host before emitting portable bitcode so that the online JIT on android devices can be light-weight.

Online JIT compiler, libbcc, performs target-specific optimizations and code generation and links dynamically against vendor-specific runtime library funcitons (lib*.bc). An example: RenderScript's runtime (libclcore.bc) comes with vector operations. Xoom's libclcore will have different CPU/GPU support (VFP3-16) than Nexus S's (NEON).

