---
title: "Asynchronous Programming"
date: 2017-06-19T00:00:00+00:00
draft: false
---

### Why Care?
Browsing through job postings you often notice a job requirement along the lines of: 
> Able to write highly efficient asynchronous code.  

A fleeting thought:

"Ah this just means writing efficient code in terms of Big-O and then split and farm out work to it by using a solution (language feature, library, framework, ...) someone else has already come up with."

You skip over this requirement without much more thought.  A few days later you are in an interview and get asked the seminal "What's the difference between a process and a thread?".   Huh? Why is that a relevant question?  You already know processes and threads are low level OS stuff.  We don't program in assembly language any more - why be concerned with OS primitives?

Writing efficient asynchronous code does have the prerequisite that your code is efficient and you are skilled at using pre-existing code.  But there is (at least) one additional skill required here: choosing and effectively implementing the asynchronous paradigm that fits the problem you are trying to solve.  When the interviewer asked you about process vs thread they were quickly checking the tip of the iceberg of what they hope you already know about asynchronous programming.  Hopefully you know this stuff, otherwise in the design interview you will be yielding blank stares instead of solutions.

### Problem Space

So what is the high level problem to be solved with the skill of writing "Efficient Asynchronous Code"?  Just writing efficient code in terms of algorithmic complexity will optimize resource usage: CPU cycles and / or memory.  The addition of "Asynchronous" implies that different parts of your code may more freely execute when needed, less constrained by their location in the source code file.

The asynchronous aspect is expected to yield scalar benefit because all it considers is when code executes given the same input, using the same algorithms.  In contrast, the goal of algorithmic efficiency tuning is an asymptotic benefit given increasing input size.  So why bother with a mere scalar increase?  At some point processing takes a minimum amount of time.  When you have many tasks taking some non-trivial amount of time, the ability to order and potentially parallelize them can have a significant impact on overall (calendar) run time.

There are 2 categories of what causes calendar wait times:

1.  CPU Bound - Time required to process data
    *  Can alternatively be addressed by a more efficient algorithm
2.  I/O Bound - Time required to move data
    *  Can alternatively be addressed by caching

Skillful application of asynchronous processing technique allows you to parallelize such time bound tasks to reduce overall calendar wait time as much as possible.  In the world of a web service platform scalar gains (2X, 3X etc) would be seen as phenomenal wins.  Even marginal gains (10%) are [valuable](https://blog.kissmetrics.com/loading-time/).


### Building Blocks
That interview question about process vs thread is checking the relationship between 2 nodes in a knowledge graph required to select the building blocks required for an overall solution.

In a nutshell:  The operating system creates and manages processes, each having it's own address space.  The operating system may also create additional threads within a process (this is generally done using the language of your choice interfacing with the OS).  Threads share the address space of the process, allocating off the same heap and also able to share references to arbitrary memory in the address space.  Some languages additionally support [coroutines](https://en.wikipedia.org/wiki/Coroutine) which have their own stack but which execute within a thread.  This leads to 3 choices each of where to allocate execution and data: Processes, Threads, or Coroutines.

![Process vs Thread](/mySiteStatic/images/BuildingBlocks_ProcessThread.png)

Of course this diagram is a simplification only showing a small subset of possibilities for the instances of {Process, Thread, Coroutine} that could exist.  For example there could be many processes, processes can spawn child processes, and there can be many instances of coroutines.  Only stacks are shown but of course other local data such as the program counter for each executable instance would need to exist.  Additionally different computing environments have different setups; e.g. a particular programming language implementation may manage threads instead of the OS (known as "green" threads).  Always read the docs (and blogs, tutorials, etc) related to your particular environment.

So how do you decide where to allocate processing?  Here are some pros and cons to consider for each:

1.  Multiple processes

    Pros

    *  True parallelism, will work for increasing overall performance of CPU bound tasks.
    *  Separate address space for each process instance, no need to worry about resource contention within the program (external resource contention may still exist, e.g. multiple processes writing to a single file).
    *  Easiest to reason about.  Only 1 entry point for execution.
    *  For \*nix platform easiest to reuse as a modular component with OS provided IPC mechanisms (eg combining small programs with pipe '|' on the command prompt).

    Cons

    *  Heavy weight.  Each process has it's own copy of program data, etc.  In addition to memory processes generally use other limited OS resources more heavily than the other options.


2.  Multiple threads

    Pros

    *  Lighter weight than processes.  Threads share the data and heap portion of process memory.
    *  May offer true parallelism for CPU bound tasks.  Check your computing environment implementation docs to be sure.

    Cons

    *  Resource contention within the process address space may be complicated to deal with, especially if the threads can be run in parallel and / or are scheduled by the OS.
    *  Difficult to reason about because a thread may be suspended at any time and have another thread change it's environment.


3.  Coroutines

    Pros

    *  Explicit control of when code is executing and when it is not executing, easier to reason about because you don't have to account for suspension in every possible location.
    *  Language support reduces complexity of gathering results.  Generally the result replaces the task in your code.  i.e. a list of tasks becomes a list of results from those tasks; no need to implement code to collect results (ie callbacks, global data structures, etc).

    Cons

    *  Does not offer local parallelism.  Only 1 coroutine may be running at a given time in the thread it shares with other coroutines.
    *  Requires implementation discipline.  It is easy to introduce blocking code which freezes your thread and subsequently all other coroutines on the thread for that duration.  Blocking code could even be part of a 3rd party library - the libraries you use need to be compatible!

Once the nature of the tasks (IO or CPU bound?) are identified along with the usability features desired for the code some combination of these options should surface as a winner.

Here are two code examples (at least Python 3.5 is required) to illustrate the execution difference between threads and coroutines.  Specifically these examples illustrate how coroutine execution is more explicit and deterministic as opposed to the willy-nilly execution that threads enjoy (but the rest of us don't). Thread and process execution is similar (both willy-nilly) so a process example is not included here.  Each example contains 3 kinds of tasks.  A CPU bound task, 4 IO bound tasks of varying lengths, and a polling task that wants to recur.

First, multi threading:
```python
from random import sample
from time import sleep, time
from threading import Thread

def io_bound(s, label="IO bound"):
    print(label + " started")
    sleep(s)
    print(label + " finished")

def cpu_bound(s, label="CPU bound"):
    print(label + " started")
    end = time() + s
    while time() < end:
        sample(range(1000), 1000)
    print(label + " finished")

child_threads = [Thread(target=io_bound, args=(s, str(s) + "s IO Bound")) for s in range(4)]
child_threads.append(Thread(target=cpu_bound, args=(2, "2s CPU Bound")))

print("Start child threads")
for t in child_threads:
    t.start()

# We are in a main thread, so this is the equivalent of the poll task from async_example
while True:
    print("Poll")
    live_threads = [t for t in child_threads if t.is_alive()]
    if len(live_threads) > 0:
        sleep(0.5)
    else:
        break

print("Main thread completed")
```

Output:
```bash
Start child threads
0s IO Bound started
0s IO Bound finished
1s IO Bound started
2s IO Bound started
3s IO Bound started
2s CPU Bound started
Poll
Poll
1s IO Bound finished
Poll
Poll
2s CPU Bound finished
2s IO Bound finished
Poll
Poll
3s IO Bound finished
Poll
Main thread completed
```

Here all threads are running independently spread throughout time.  The drawback is seen when considering the CPU bound task.  While it is running other threads are allowed to run.  It can be preempted mid operation and have things change under it's feet.  If this task needed to access resources also accessible to the other tasks then potentially complex synchronization implementation is required.  If for example it was placing partial results in a shared global location there would need to be additional code to synchronize access to that location across threads.  One positive aspect here is shown by the polling task: it is allowed to run without being starved by the CPU bound task.

Now the asynchronous version using asyncio - Python's coroutine support.

```python
import asyncio
from random import sample
from time import time

async def io_bound(s, label="IO bound"):
    print(label + " started")
    await asyncio.sleep(s)
    print(label + " finished")

# Example of a blocking task
async def cpu_bound(s, label="CPU bound"):
    print(label + " started")
    end = time() + s
    while time() < end:
        sample(range(1000), 1000)
    print(label + " finished")

async def poll(s):
    while True:
        print("Poll")
        await asyncio.sleep(s)
        active_tasks = [task for task in asyncio.Task.all_tasks() if not task.done()]
        if  len(active_tasks) < 3: # this poll() and wait() for run_until_complete will always exist
            return

loop = asyncio.get_event_loop()
tasks = [asyncio.ensure_future(io_bound(s, str(s) + "s IO Bound")) for s in range(4)]
tasks.append(asyncio.ensure_future(cpu_bound(2, "2s CPU Bound")))
tasks.append(asyncio.ensure_future(poll(.5)))

loop.run_until_complete(asyncio.wait(tasks))

print("Event loop completed")
```

Output:
```bash
0s IO Bound started
1s IO Bound started
2s IO Bound started
3s IO Bound started
2s CPU Bound started
2s CPU Bound finished
Poll
0s IO Bound finished
1s IO Bound finished
2s IO Bound finished
Poll
3s IO Bound finished
Event loop completed
```

The difference here is that the CPU bound tasks runs to completion without being interrupted.  You may ask yourself - isn't this a bad thing?  Yes it is for CPU bound tasks and that's why you shouldn't use coroutines to parallelize those.  But it illustrates the point that your code will not be interrupted willy-nilly!  Since execution won't be preempted you don't have to deal with implementing a synchronization strategy for shared resources.  All the tasks that have to wait for _external_ IO bound processing to complete effectively have the work processed in parallel - slashing the overall wait time.  All of this is done very efficiently and without the headache of resource synchronization!  Waiting on IO bound external tasks is very common in web development, which is why this form of asynchronous processing has become all the rage.
