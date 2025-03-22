#### Why Concurrency

Suppose you are on on main thread and you need data from server. You requested data from server and waits until you get the response from server. During this duration your main thread will not perform any UI related work which makes your application unresponsive. 

#### Concurrency

Concurrency means that an application is making progress on more than one task at the same time using time slicing.

If the computer only has one CPU, then the application may not make progress on more than one task at exactly the same time

but more than one task is being processed at a time inside the application using the technique called context switching.

For the case we discussed if we execute network call using Concurrency then there will be two threads,main and background executing instruction(操作) using context switching

[In computing, a **context switch** is the process of storing the state of a process or of a thread, so that it can be restored and execution resumed from the same point later. This allows multiple processes to share a single CPU, and is an essential feature of a multitasking operating system.](https://en.wikipedia.org/wiki/Context_switch)

#### **Grand Central Dispatch (GCD)**

**GCD** is a low-level API for doing Concurrency / Parallelism in your application.

#### How GCD perform Concurrency / Parallelism

GCD  manage a **shared thread pools** and add optimal（最佳的） number of threads in that pool.

With GCD you add blocks of code or work items to queues and GCD decides which thread to execute them on.

**Note:** If you give two tasks to GCD you are not sure whether it will run concurrently or parallely.

#### Without GCD

In the past we do concurrency using manually creating threads,Threaded solutions is a low level solutions and must be managed manually.

* Synchronization mechanisms（机制） typically used with threads add complexity

#### Developer Responsibility With GCD

All you have to do is define the tasks you want to execute concurrently and add them to an appropriate dispatch queue. GCD takes care of creating the needed threads and of scheduling your tasks to run on those threads super cool 😃

#### GCD Benefits

1. It provide you simple programing interface. (Swift)
2. Automatic thread pool management
3. Don’t trap kernel under the load
4. Dynamic thread scaling on the basis of current system loads
5. Memory efficient since thread stacks don’t in application memory it is in system memory
6. Better use cores efficiently because these logic moves to system level



#### Dispatch Queues

Dispatch queues are thread-safe which means that you can access them from multiple threads simultaneously（同时地）. **Dispatch** **Queue is not Thread**

Dispatch queue is the core of the GCD. On the basis of Dispatch queues configuration GCD pick and execute concurrent tasks.

#### Serial Dispatch Queue

1. Execute one task at a time in the order in which they are added to the queue. Let’s say if you added five tasks to serial configured dispatch queue GCD will start from first task and execute it until the first task is not completed it will not pick second task and so on
2. Serial queues are often used to synchronize access to a specific resource.
3. if you create four serial queues, each queue executes only one task at a time but up to four tasks could still execute concurrently, one from each queue.
4. If you have two tasks that access the same shared resource but run on different threads, either thread could modify the resource first and you would need to use a lock to ensure that both tasks did not modify that resource at the same time. With dispatch queues, you could add both tasks to a serial dispatch queue to ensure that only one task modified the resource at any given time. This type of queue-based synchronization is more efficient than locks because locks always require an expensive kernel trap in both the contested and uncontested cases, whereas a dispatch queue works primarily in your application’s process space and only calls down to the kernel when absolutely necessary.

#### Concurrent Dispatch Queue

1. Concurrent queues (also known as a type of global dispatch queue) execute one or more tasks concurrently,
2. If you added four separate tasks to this global queue, those blocks will starts tasks in the same order in which they were added to the queue. GCD pick first task execute it during some time and then start second tasks without waiting for first task to complete and so on. 
3. The currently executing tasks run on distinct threads that are managed by the dispatch queue.
4. In GCD, there are two ways I can run blocks concurrently either creating custom concurrent queue or using global concurrent queue

#### Difference b/w Custom & Global Concurrent Queue

As shown in Figure 1 we created two concurrent queues and you can see since **Global queques** are Concurrent queues that are shared by the whole system it always return the same queue whereas custom concurrent queue is private returning new queue every time you create it.

There are four global concurrent queues with different priorities When setting up the global concurrent queues, you don’t specify the priority directly. Instead you specify a Quality of Service (QoS) which includes User-interactive,User-initiated, Utility and Background with the User-interactive has highest priorities and Background with the least. You can see when to use from this [**link**](https://medium.com/@crafttang/ios-grand-central-dispatch-cf9b08d9796b)

#### Main Dispatch Queue

1. The main dispatch queue is a globally available serial queue that executes tasks on the application’s main thread
2. This queue works with the application’s run loop (if one is present) 

#### Synchronous vs. Asynchronous

In general **synchronous** function returns control to the caller after the task completes if you dispatch queue sync until and unless all the tasks in the queue completed and queue is empty it will not return to the caller.

In contrast **asynchronous** function returns immediately to the caller before the task completes if you dispatch queue async it will not waits for the tasks in the queue executes or not， and immediately return to the caller











































