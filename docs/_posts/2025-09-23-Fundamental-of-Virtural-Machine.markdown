---
layout: post
title:  "Fundamental of Virtural Machine"
date:   2025-09-23 11:01:11 +0900
categories: Virtual Machine
---


I spend some time to read this article [fundamental_of_virtual_memory](https://nghiant3223.github.io/2025/05/29/fundamental_of_virtual_memory.html) in the day. 

Here I records some points that I don't know before , along with some derived idea base on the article.

1. All the variables whose size can be determined presiely at compile time will be allocated to the stack. 
2. All the variable allocated on the stack can be reclaimed efficiently by simply moving  the stack pointer down. 
3. From 1,2 allocate or reclaimed variables on the stack is very fast.
4. However, in some cases, variables must be allocated on the heap instead of stack: 
   4.1 When the variable type or size cannot be determined at compile time.
   4.2 When the method A is called by multiple methods (e.g B and C). In such cases, the variables of method A cannot remain on the stack,  because they would be lost once A's stack frame is popped. Therefore,they must be stored on the heap.
5. A process can create multiple threads. Each thread will have it's own stack, but these threads share the heap. 
6. Each thread's stack has size limit, which is defined by the operating system, but can also be adjusted by the virtural machine.
7. The default stack size per thread is typically 2MB. we use this to estimate the maximum number of threads a VM can support,  max thread count = (available memory / thread memory size). However, it's not clear whether the actual memory usage is always affected by this allocation, since OS may reserve 2 MB per thread regardless of real consumption.
8. In addition to 7, we also need to consider heap usage when estimating the maximum supported thread count. Is there a straighforward way to determin this factor, so that we can better understand the pratical maximum concurrency ? 


