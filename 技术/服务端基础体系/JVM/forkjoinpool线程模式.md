#### forkjoin定义
_forkjoinpool定义_
```
A ForkJoinPool differs from other kinds of ExecutorService mainly by virtue of employing work-stealing: all threads in the pool attempt to find and execute tasks submitted to the pool and/or created by other active tasks (eventually blocking waiting for work if none exist). This enables efficient processing when most tasks spawn other subtasks (as do most ForkJoinTasks), as well as when many small tasks are submitted to the pool from external clients. Especially when setting asyncMode to true in constructors, ForkJoinPools may also be appropriate for use with event-style tasks that are never joined.

从oracledoc上看到该定义，区别与其它类型的线程池，主要采用了work-stealing模式。这种模式对于任务依赖其它子任务或其它很多小任务模式更为高效。同样也可能适用于我们常规的非join的异步模式。
```

_forkjoin问题域_
![](../picture/forkjoin.png)

从上图中可以看到，fork/join模式，主要解决的分治问题。具体分治算法参考：[分治维基百科](https://zh.wikipedia.org/wiki/%E5%88%86%E6%B2%BB%E6%B3%95)

#### work-stealing
```
在工作窃取调度程序中，计算机系统中的每个处理器都有一个要执行的工作项（计算任务，线程）队列。每个工作项都包含一系列要顺序执行的指令，但是在其执行过程中，一个工作项也可能产生新的工作项，这些新工作项可以与其他工作并行执行。这些新项目最初放在执行工作项目的处理器的队列中。当处理器的工作量用尽时，它将查看其他处理器的队列并“窃取”其工作项。实际上，工作窃取会将调度工作分配到空闲的处理器上，并且只要所有处理器都有工作要做，就不会发生调度开销。

工作窃取与工作共享形成了对比，工作共享是动态多线程的另一种流行的调度方法，其中，每个工作项在生成时都被调度到处理器上。与这种方法相比，工作窃取减少了处理器之间的进程迁移量，因为当所有处理器都有工作要做时，不会发生这种迁移。
```
上述是维基百科上的解释，可以看出来work-stealing是一个线程共享一个任务队列。工作共享是每一个任务都产生一个线程，处理器来决定处理哪一个任务。