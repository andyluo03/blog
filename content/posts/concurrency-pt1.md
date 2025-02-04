+++
title = "Concurrency Under The Hood Part 1: Concurrency vs. Parallelism"
date = 1738647783
draft = false
disqus = true
+++

# Background

As part of building a [(crappy) HTTP server](https://github.com/andyluo03/brick) capable of handling 10k 
concurrent connections, I discovered that, at the heart of the problem, was... concurrency!

However, online resources were primarily targeted at new web developers without much care about the internal workings.
As such, I had a difficult time finding sources for building my [own concurrency engine](https://github.com/andyluo03/Vial). 
Thus, I wrote this post in hopes it will help future programmers understand what their concurrent code is really doing. 

This article aims to be accessible by beginners but consumed by intermediate+ skill levels. Unfortunately, as this is a complex
topic, I am assuming readers understand the basics of threading. However, later blogs will likely only be accessible by 
intermediate to advanced readers.

I will not be covering *how* to write concurrent programs including topics: like memory barriers, locks, etc.

# Key Definitions 

| Vocab | Definition |
| --- | --- |
| Parallelism | Executing multiple tasks simultaneously (via multiple CPU cores, GPU cores, etc.) |
| Concurrency | Handling multiple tasks that progress independently by interleaving their execution (could be single or multi-threaded) | 

# Real World Examples of Concurrency

| Name | How it leverages concurrency | 
| --- | --- | 
| Operating Systems | OS threads are managed by a scheduler to help improve fairness and ensure all resources are being used. |
| IO-Bound: Tokio (Rust), Async/Await (Python) | The asyncio model allows tasks that aren't waiting for IO to execute. This is particularly useful in cases such as 10k+ concurrent connections where a particularly slow connection might otherwise clog up your server! |
| CPU-Bound: Rayon (Rust), Intel TBB (C++)| These parallel execution frameworks utilize schedulers to handle concurrent tasks on top of a thread pool. This strategy helps reduce costs (but does not completely avoid) associated with OS context switches. |

# Concurrency vs. Parallelism

While the two often go hand-in-hand, Parallelism != Concurrency. it's okay if you don't fully grasp the differences at this point!
We're about to cover some examples to (hopefully) clear up any confusions you might have. 

## Concurrency without Parallelism

Let's define a simple single-threaded multiplexer as the following state and rules. Here, a multiplexer
is simply a way to interleave tasks to avoid downtime on a single thread. This concept will serve as 
building blocks for our later concurrency models. 

| State | Description |
| --- | --- |
| Queue | Iterable container for tasks |

Rules: 
```
Check Queue Empty:
    Empty: Terminate

Iterate over Tasks in Queue:
    If Task Ready:
        Execute Task:
            If Task Complete:
                Remove Task from Queue
            If Task Awaiting:
                Add task being awaited on to Queue

Repeat
```

Next, take this python code:

```py
1.  async def A():
2.     return 1 + await B()
3.
4.  async def B():
5.     return 1 + await C()
6.
7.  async def C():
8.     return 1
9.
10. async def main (): 
11.    await A()
12.
13. multiplexer.run(main())
```

So what happens when we run this in a single-threaded context? 

```
Iteration 1:
main awaits A               | queue: [main, A]
A ready --> awaits B        | queue: [main, A, B]
B ready --> awaits C        | queue: [main, A, B, C]
C ready --> complete!       | queue: [main, A, B]

Iteration 2:
main not ready, awaiting A      | queue: [main, A, B]
A not ready, awaiting B         | queue: [main, A, B]
B ready --> complete!           | queue: [main, A]

Iteration 3:
main not ready, awaiting A      | queue: [main, A]
A ready --> complete!           | queue: [main]

Iteration 4:
main ready --> complete!      | queue: []

DONE!
```

Voila! We have now properly handled MULTIPLE TASKS on a SINGLE THREAD (if you want to see a more robust example I encourage you to draw out some more scenarios). In total, it took ~10 pseudo-instructions.

## Parallelism without Concurrency

Unlike concurrency, parallelism is a little bit less complicated (for now...) The key idea is that we 
can utilize multiple CPU cores to do tasks *at the same time* to get some performance gain. 

Let's take a look at this multi-threaded pseudocode:

```
fn sum(array):
    int x = 0
    for i in array:
        x += i
    return x

def main():
    arrays = [[1, 2, 3], ..., [9, 1234, 3]] // 8 arrays
    threads = []

    for i in arrays:
        threads.push( thread(sum(i)) ) # Spawns a thread that runs sum(i)
    
    for i in threads:
        i.join() # Wait for thread i to complete
```

This should be pretty easy to understand. We receive 8 arrays, spawn threads to add up each of them, and then wait for all of them to finish. 
This could give us up to an 8x performance boost on the multi-threaded portions (look into Amdahl's law) however in a more realistic example,
we would have costs associated with hardware, context switching, etc.

Please note that your OS will almost definitely manage your threads so there is *some* form of concurrency going on. 

## Putting it all together

Now that we understand the two concepts separately, let's take a look at how they go together! 

First, let's define a threadpool. A threadpool is simply a group of OS threads that can be reused for multiple tasks.
This will be how we achieve parallelism without spawning too many OS level threads!

First, let define our multi-threaded multiplexer:

| State | Description |
| --- | --- |
| Queue | FIFO container for tasks |

Next, we will have the thread pool follow similar rules to the single-threaded multiplexer:

```
Take the first task from Queue

If Task Ready:
    Execute Task:
        If Task awaiting:
            Push task being awaited on to Queue
            Push Task to Queue
Else:
    Push Task to Queue
```

Now let's imagine the same previous pseudocode:
```py
1.  async def A():
2.     return 1 + await B()
3.
4.  async def B():
5.     return 1 + await C()
6.
7.  async def C():
8.     return 1
9.
10. async def main (): 
11.    await A()
12.
13. multiplexer.run(main())
```

Let's do a run with TWO threads in the threadpool! (note the state is AFTER the instruction is run)

```
Thread 1 | main awaits A             | Queue: [A, main]
Thread 2 | A ready --> awaits B      | Queue: [B, main, A]

Thread 1 | B ready --> awaits C      | Queue: [C, main, A, B]
Thread 2 | C ready --> complete!     | Queue: [main, A, B]

Thread 1 | main not ready            | Queue: [A, B, main]
Thread 2 | B not ready               | Queue: [B, main, A]

Thread 1 | B ready --> complete!     | Queue: [main, A]
Thread 2 | main not ready            | Queue: [A, main]

Thread 1 | A ready --> complete!     | Queue: [main]
Thread 2 | main ready --> complete!  | Queue: []

DONE!
```

Notice how, although we have the same amount of pseudo-instructions (10), we split the work across two threads giving us
a 2x speed up as each CPU core only needs to execute 5 pseudo-instructions. (Note that we are not considering costs of
synchronization, context switches, hardware overhead, etc.) 

## Final Notes

Concurrency is not just about running multiple tasks, but rather, it is about structuring execution of tasks efficiently. 
Understanding when to utilize concurrency, parallelism, and both is key to architecting scalable sofware. 

After reading this, you should be able to explain the difference between concurrency and parallelism as well as how they go 
together. You should also be able to decide which concurrency framework will best suit your purposes for your own projects.

The concurrency model we covered is called "cooperative multitasking." Other models such as preemptive multitasking (where
the scheduler interrupts tasks rather than functions yielding/awaiting) also exist. This will be further developed in future 
posts covering stackful vs. stackless coroutines!

The solutions we looked at for concurrency, parallelism, and combinations ARE NOT optimal and will be improved upon
in later blog posts. 

The next blog post(s) will discuss various scheduling strategies and their trade-offs (work-stealing, LIFO, FIFO) as well as dive into
some more real-world examples including better benchmarks. 
