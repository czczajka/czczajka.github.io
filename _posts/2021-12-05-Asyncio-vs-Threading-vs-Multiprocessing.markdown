---
layout: post
title:  "Asyncio vs Threading vs Multiprocessing"
date:   2021-12-05 00:12:24 +0100
---

## List of contents:
1. Threading
2. Asyncio
3. Multiprocessing

### Threading
#### GIL
Global interpreter lock is the mechanism used by the CPython interpreter to assure that only one thread executes Python bytecode at a time. Python offers to switch among threads only between bytecode instructions (how frequently it switches can be set via sys.setcheckinterval()).
The main idea of GIL is that all created and started threads are run on the single core. In CPython is not possible to run threads on a diffrent cores. Only one core can be utilize in maximum.

# sys.setswitcinterval()
sys.setswitchinterval() method is used to set the interpreter’s thread switch interval (in seconds). This floating-point value determines the ideal duration of the timeslices allocated to concurrently running Python threads. The actual value can be higher, especially if long-running internal functions or methods are used. Also, which thread becomes scheduled at the end of the interval is the operating system’s decision. The interpreter doesn’t have its own scheduler. This determines how often the interpreter checks for thread switches.

# Daemon Threads

# join()
If, for example, you want to concurrently download a bunch of pages to concatenate them into a single large page, you may start concurrent downloads using threads, but need to wait until the last page/thread is finished before you start assembling a single page out of many. That's when you use join().

In multiprocessing you leverage multiple CPUs to distribute your calculations. Since each of the CPUs runs in parallel, you're effectively able to run multiple tasks simultaneously. You would want to use multiprocessing for CPU-bound tasks. An example would be trying to calculate a sum of all elements of a huge list. If your machine has 8 cores, you can "cut" the list into 8 smaller lists and calculate the sum of each of those lists separately on separate core and then just add up those numbers. You'll get a ~8x speedup by doing that.

In (multi)threading you don't need multiple CPUs. Imagine a program that sends lots of HTTP requests to the web. If you used a single-threaded program, it would stop the execution (block) at each request, wait for a response, and then continue once received a response. The problem here is that your CPU isn't really doing work while waiting for some external server to do the job; it could have actually done some useful work in the meantime! The fix is to use threads - you can create many of them, each responsible for requesting some content from the web. The nice thing about threads is that, even if they run on one CPU, the CPU from time to time "freezes" the execution of one thread and jumps to executing the other one (it's called context switching and it happens constantly at non-deterministic intervals). So if your task is I/O bound - use threading.

asyncio is essentially threading where not the CPU but you, as a programmer (or actually your application), decide where and when does the context switch happen. In Python you use an await keyword to suspend the execution of your coroutine (defined using async keyword)\

The asyncio package is billed by the Python documentation as a library to write concurrent code. However, async IO is not threading, nor is it multiprocessing. It is not built on top of either of these.

In fact, async IO is a single-threaded, single-process design: it uses cooperative multitasking, a term that you’ll flesh out by the end of this tutorial. It has been said in other words that async IO gives a feeling of concurrency despite using a single thread in a single process. Coroutines (a central feature of async IO) can be scheduled concurrently, but they are not inherently concurrent..
