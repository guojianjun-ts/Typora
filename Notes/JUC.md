# Java 并发

## 线程

#### 进程和线程是什么？

1. **进程**：
   - 是操作系统分配资源的基本单位，是一个正在执行的程序的实例。
   - 每个进程都有独立的内存空间、代码、数据和系统资源。
2. **线程**：
   - 是CPU调度的基本单位，线程是进程中的一个执行单元。
   - 一个进程可以包含多个线程，这些线程共享进程的内存空间和资源。

#### Java线程和操作系统的线程是否一样？

1. 在 JDK 1.2 之前：
   - Java 线程是基于绿色线程（Green Threads）实现的，这是一种用户线程，也就是说 JVM 自己模拟了操作系统中多线程的运行，不依赖于操作系统。
   - 缺点：绿色线程不能直接使用操作系统提供的功能: 异步I/O，且只能在一个内核线程上运行，无法利用多核。
2. 在 JDK 1.2 及之后：
   - Java 线程改成了基于原生线程（Native Threads）实现，这是一种内核线程，也就是说 JVM 可以直接使用操作系统原生的内核线程来实现 Java 线程，由操作系统自己调度和管理线程。
   - 所以现在来说，Java 线程的本质就是操作系统的线程。
   - 用户线程和内核线程的区别和特点：用户线程创建和切换成本低，但不可以利用多核。内核态线程，创建和切换成本高，可以利用多核。

#### 简要描述线程和进程的关系，区别和优缺点

|   **特性**   |                **进程**                |                    **线程**                     |
| :----------: | :------------------------------------: | :---------------------------------------------: |
|   **定义**   |       操作系统分配资源的基本单位       |       进程中的执行单元，CPU调度的基本单位       |
| **资源分配** |          独立的内存空间和资源          |            共享进程的内存空间和资源             |
|   **开销**   |      :no_entry:创建和销毁开销较大      |               创建和销毁开销较小                |
| **通信方式** | :no_entry:进程间通信（IPC）机制 (复杂) |            直接共享内存，通信更高效             |
|  **独立性**  |            进程之间相互独立            | :no_entry:线程之间共享资源，相互依赖 (死锁问题) |
| **崩溃影响** |      一个进程崩溃不会影响其他进程      |   :no_entry:一个线程崩溃可能导致整个进程崩溃    |
| **适用场景** |    需要隔离的任务（如不同应用程序）    |       需要并发执行的任务（如多任务处理）        |

#### 创建线程的几种方式？

- **表象上**：Java提供了多种创建线程的方式，如继承 `Thread`、实现 `Runnable` 、`Callable` 接口、使用线程池等、使用 `CompletableFuture` 等。

- **本质上**：Java创建线程的方式只有一种，那就是通过 `Thread.start()` 方法来启动线程。而所谓`Runnable` 、`Callable`…… 等对象，这仅仅只是线程体，也就是提供给线程执行的任务，并不属于真正的 `Java` 线程，它们的执行，最终还是需要依赖于`new Thread()`……

- 大致过程：

  **① 类加载阶段：绑定JVM本地方法：** `Thread` 在类加载阶段，就会通过静态代码块去绑定`Thread`类方法与`JVM`本地方法的关系：

  ```java
  private static native void registerNatives();
  static {
      registerNatives();
  }
  ```

  执行完这个 `registerNatives()` 本地方法后， `Java` 的线程方法，就和 `JVM` 方法绑定了，如 `start0()` 这个方法，会对应着 `JVM_StartThread()` 这个 `C++` 函数等。

  **② 调用 `Thread.start()` 方法：** 当调用 `Thread.start()` 方法后，会先调用 `Java` 中定义的 `start0()` ，接着会找到与之绑定的 `JVM_StartThread()` 这个 `JVM` 函数执行（具体实现位于 `openjdk\hotspot\src\share\vm\prims\jvm.cpp`这个文件）。

  **③ 创建内核线程：** `JVM_StartThread()` 函数最终会调用 `os::create_thread(...)` 这个函数，这个函数依旧是 `JVM` 函数，毕竟 `Java` 要实现跨平台特性，而不同操作系统创建线程的内核函数，也有所差异，如 `Linux` 操作系统中，创建线程最终会调用到 `pthread_create(...)` 这个内核函数。

  **④ 映射Java线程与内核线程：**创建出一条内核线程后，接着会去执行 `Thread::start(...)` 函数，接着会去执行`os::start_thread(thread)` 这个函数，这一步的作用，主要是让 `Java` 线程，和内核线程产生映射关系，也会在这一步，把 `Runnable` 线程体，顺势传递给 `OS` 的内核线程（具体实现位于 `openjdk\hotspot\src\share\vm\runtime\Thread.cpp`这个文件）。

  **⑤ 执行线程体：**当 `Java` 线程与内核线程产生映射后，接着就会执行载入的线程体（线程任务），也就是 `Java` 程序员所编写的那个 `run()` 方法。

#### 怎么启动线程？

1. 正确方式：

   - 在Java中，启动线程的唯一方式是调用 **`Thread.start()`** 方法。

2. 错误方式：

   - 直接调用 `run()` 方法（不会启动新线程）。

   - 重复调用 `start()` 方法（会抛出异常）。

#### 如何停止一个线程的运行？	

|       **方法**        |         **适用场景**         |     **优点**     |          **缺点**          |
| :-------------------: | :--------------------------: | :--------------: | :------------------------: |
|    `Thread.stop()`    |         强制终止线程         |     简单直接     | 已被弃用，可能导致资源泄漏 |
|      标志位控制       |   需要安全、可控地停止线程   |    安全、可控    |   需要线程主动检查标志位   |
| `Thread.interrupt()`  |      需要优雅地停止线程      | 适合处理阻塞操作 |  需要线程主动检查中断状态  |
|    `Thread.join()`    |       需要等待线程结束       |     简单易用     |    只能等待线程自然结束    |
| 线程池的 `shutdown()` |      停止线程池中的线程      |  适合管理线程池  |    无法立即停止所有线程    |
|   `Future.cancel()`   | 取消通过 `Future` 提交的任务 | 适合取消特定任务 |     需要结合线程池使用     |

​	在实际开发中，**推荐使用标志位控制或 `Thread.interrupt()`**，因为它们更安全、可控，且不会导致资源泄漏。对于线程池中的线程，可以使用 `shutdown()` 或 `shutdownNow()` 方法。	

#### 说说线程的生命周期和状态

Java 线程在运行的生命周期中的指定时刻只可能处于下面 6 种不同状态的其中一个状态：

1. NEW: 初始状态，线程被创建出来但没有被调用 `start()` 。
2. RUNNABLE: 运行状态，线程被调用了 `start()`等待运行的状态。
3. BLOCKED：阻塞状态，需要等待锁释放。
4. WAITING：等待状态，表示该线程需要等待其他线程做出一些特定动作（通知或中断）。
5. TIME_WAITING：超时等待状态，可以在指定的时间后自行返回而不是像 WAITING 那样一直等待。
6. TERMINATED：终止状态，表示该线程已经运行完成。

![Java 线程状态变迁图](../assets/640.png)

#### 什么是线程的上下文切换

1. 定义：上下文切换是指操作系统在切换执行线程时，保存当前线程的状态（如寄存器、程序计数器等），并加载另一个线程的状态，以便恢复其执行。
2. 上下文：是指线程执行时所需的所有状态信息，包括：
   - **寄存器**：如程序计数器（PC）、栈指针（SP）等。
   - **内存管理信息**：如页表、内存映射等。
   - **其他状态**：如线程优先级、信号掩码等。
3. **触发条件**：
   - 线程主动让出CPU（如调用 `sleep()`、`wait()` 等）。
   - 线程的时间片用完（分时调度）。
   - 更高优先级的线程需要执行（抢占式调度）。
   - 线程执行阻塞操作（如I/O操作）。

#### sleep()和wait()的区别

| **对比维度** |              **`wait()`**               |          **`sleep()`**           |
| :----------: | :-------------------------------------: | :------------------------------: |
|  **所属类**  |            `Object` 类的方法            |      `Thread` 类的静态方法       |
| **锁的释放** |           ✔️ 立即释放锁(阻塞)            |         ❌ 不释放锁(阻塞)         |
| **唤醒条件** | 需其他线程调用 `notify()`/`notifyAll()` | 休眠时间结束或调用 `interrupt()` |
| **使用场景** |      线程间协作（如生产者-消费者）      |   单纯暂停线程执行（如倒计时）   |
| **异常捕获** |      需处理 `InterruptedException`      |  需处理 `InterruptedException`   |
| **同步要求** |     必须在 `synchronized` 块中调用      |             无需同步             |

#### 直接调用 run() 会怎么样？

1. 直接调用方法 `run()` ，那么`run()` 会直接在当前线程( `main` 线程)中执行，这并不是多线程工作。那么它就只是一个普通的方法调用，不会启动新线程，线程仍然是 `RUNNABLE` 状态。
2. 通过调用 `start()` 方法启动一个新线程，这个线程会再执行 `run()` 方法，线程会由  `NEW` 变为 `RUNNABLE` 
3. 总结：

|      **特性**      |    **直接调用 `run()`**    |    **调用 `start()`**    |
| :----------------: | :------------------------: | :----------------------: |
| **是否启动新线程** |       不会启动新线程       |       会启动新线程       |
|    **执行线程**    |      在当前线程中执行      |      在新线程中执行      |
|    **适用场景**    |  普通方法调用、调试和测试  |   需要启动新线程的场景   |
|  **线程安全问题**  | 可能存在问题（单线程执行） | 需要额外处理线程安全问题 |

### 多线程

#### 并行和并发的区别？

- **并行：**是多核CPU的多任务处理，多个任务在同一时刻真正被同时处理。
- **并发：**是单个CPU的多任务处理，每个任务各占用一个很短的时间段(20~50ms)，然后在1s来看，看起来就像同时运行了多个进程，产生了并行的错觉，但其实是并发。

#### 同步和异步的区别？

- **同步**：发出一个调用之后，在没有得到结果之前， 该调用就不可以返回，一直等待。
- **异步**：调用在发出之后，不用等待返回结果，该调用直接返回。

#### 为什么使用多线程？

- 单核场景下：使用多线程主要是减少I/O操作场景下的等待时间，多线程在等待期间执行其它任务，避免阻塞主线程，提高Java进程对系统资源的整体利用率。
- 多核场景下：使用多线程主要是为了提高进程利用多核CPU的能力，允许程序同时执行多个任务，提高任务的执行效率。
- 资源利用方面：多线程可以共享进程的内存空间和资源，避免了进程间通信的开销，提高线程间的通信效率；同时线程是程序执行的最小单位，线程的创建、调度、上下文切换的开销都比进程小得多。
- 从互联网发展趋势来说：现在的系统吞吐量和并发量要求都很大很大(百万计或千万级)，多线程并发编程正式开发高并发系统的基础，使用好多线程可以大大提高系统的并发处理能力和吞吐量。

#### 单核CPU支持多线程吗？

单核CPU也是支持多线程的，操作系统通过适当的线程调度方式和上下文切换机制来支持多线程，通过快速在线程中切换模拟出并发执行的效果。

#### 单核CPU上运行多个线程就一定效率高吗？

​	不一定的，分为两个类型的线程：

1. CPU密集型：多个线程同时运行会导致频繁的线程切换，增加了系统的开销，降低了效率。
2. I/O密集型：多个线程同时运行可以利用 CPU 在等待 IO 时的空闲时间，提高了效率。

### 调度

#### 调度时机：

1. 就绪状态-->运行态
2. 运行态-->阻塞态
3. 运行态-->结束态

#### 调度原则：

- CPU 利用率：调度程序应确保 CPU 是始终匆忙的状态，这可提高 CPU 的利用率；
- 系统吞吐量：吞吐量表示的是单位时间内 CPU 完成进程的数量，长作业的进程会占用较长的 CPU 资源，因此会降低吞吐量，相反，短作业的进程会提升系统吞吐量；
- 周转时间：周转时间是进程运行+阻塞时间+等待时间的总和，一个进程的周转时间越小越好；
- 等待时间：这个等待时间不是阻塞状态的时间，而是进程处于就绪队列的时间，等待的时间越长，用户越不满意；
- 响应时间：用户提交请求到系统第一次产生响应所花费的时间，在交互式系统中，响应时间是衡量调度算法好坏的主要标准。

#### 调度算法：

1. 非抢占式调度算法：是指一旦线程获得CPU，就会一直执行，直到线程主动放弃CPU（阻塞或结束)。
   1. **先来先服务（First Come First Serve, FCFS）**
      - 按照线程到达就绪队列的顺序进行调度，先到的线程先执行。
   2. **最短作业优先（Shortest Job First, SJF）**
      - 优先调度执行时间最短的线程。
   3. **高响应比优先 （Highest Response Ratio Next, HRRN）**
      - 选择响应比最高的线程执行。
      - 响应比的计算公式为：**响应比 = $\frac{\text{等待时间} + \text{执行时间}}{\text{执行时间}}$**
2. 抢占式调度算法：是指操作系统可以强制暂停正在执行的线程，将CPU分给其他线程。(时间片轮转或I/O完成)
   1. **时间片轮转（Round Robin, RR）**
      - 每个线程被分配一个固定的时间片（Time Slice），当时间片用完时，线程被暂停并放到就绪队列末尾，调度下一个线程执行。
   2. **最高优先级（Highest Priority First，HPF）**
      - 优先调度优先级最高的线程。高优先级线程可以抢占低优先级线程的CPU时间。
   3. **多级反馈队列（Multilevel Feedback Queue）**
      - 将线程分为多个队列，每个队列结合不同的调度策略（时间片轮转、优先级调度)，优先级高的时间片越短

### 多线程带来的问题

​	并发编程虽然可以提高程序的执行效率和资源利用率，但是也会带来一些额外的问题：

1. **线程安全问题：**多个线程共享资源时，可能会引发竞态条件、死锁等问题
2. **调式和测试：**并发程序的调试和测试比单线程程序更复杂，因为线程的执行顺序是不确定的。
3. **资源消耗：**虽然线程比进程轻量，但创建过多线程仍会消耗大量内存和CPU资源，可能导致系统性能下降。

#### 如何理解线程安全和线程不安全

1. 线程安全与不安全主要指在多线程场景下，对于同一份数据是否能够保证它的正确性和一致性。


### Java '锁'事

首先：依据特性对 Java 锁进行分组分类

<img src="../assets/7f749fc8.png" alt="img" style="zoom:50%;" />

#### 1. 乐观锁or悲观锁

1. 悲观锁：对于同一个数据的并发操作，悲观锁认为自己在使用数据的时候一定会有别的线程来修改数据，因此它在获取数据的时候先加锁，确保数据不会被别的线程修改。Java 中，synchronized 关键字和 Lock 实现的都是悲观锁。
2. 乐观锁：对于同一个数据的并发操作，乐观锁认为自己在使用数据的时候不会有别的线程来修改数据，所以它不会添加锁，只是在更新数据的时候去判断数据有没有被其它的线程所修改。如果数据没有被修改，那么当前线程将修改好的数据成功写入；如果发现数据被修改了，那么就根据不同的实现方式来执行不同的操作（例如报错或重试）
3. 悲观锁基本都是在显式的锁定之后再操作同步资源，而乐观锁则直接去操作同步资源，这是因为乐观锁是由无锁编程来实现的，通过CAS算法。
4. 使用场景：
   - 悲观锁适合写多的场景，先加锁可以保证数据的正确性和一致性。
   - 乐观锁适合读多的场景，不加锁的特点可以使得其读操作的性能大幅度提升。

##### 1.1 那么为什么乐观锁能做到不锁定同步资源就可以正确的实现线程同步呢？

​	这是因为乐观锁的主要实现方式为 CAS 算法

##### 1.2 CAS算法

- 概述：CAS 全称 Compare And Swap（比较与交换），是一种无锁算法。在不使用锁（没有线程被阻塞）的情况下实现多线程之间的变量同步。

- 三个操作数：

  - 需要读写的内存值 V

  - 进行比较的值 A

  - 要写入的值 B

- 操作过程：当且仅当 V 的值等于 A时，CAS 通过原子方式用新值 B 来更新 V 的值(“比较”和“更新”整体是一个原子操作)，否则不会执行任何操作，"更新"是个不断重试的操作。

- 三大问题：

  - ABA问题：CAS在检查内存值是否发生变化时，出现了特殊情况：A-B-A，这时CAS检查会发现没有发生任何的变法，但其实是发生了变化的
    - ABA问题解决方案：在变量前添加版本号，每次更新的时候都把版本号+1，特殊情况的变化过程由“A-B-A”变为了"1A-2B-3A"
  - 循环时间长开销大问题：CAS操作如果长时间不成功，会导致其一直自旋，给CPU带来非常大的开销
  - 只能保证一个共享变量的原子操作：对一个共享变量执行操作时，CAS可以保证其原子性，但是对多个共享变量执行操作时，CAS不可以保证它的原子性
    - Java从1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作。

#### 2. 自旋锁 VS 适应性自旋锁

##### 2.1 为什么要有自旋锁？

​	首先，阻塞或唤醒一个Java线程需要操作系统切换CPU来完成，这样的状态转换需要耗费处理器时间。

​	但是，如果同步代码块中的内容过于简单，状态转换消耗的时间有可能比用户代码执行的时间还要长。而在很多场景中，同步资源的锁定耗时很短，为了这一小段时间去切换线程，线程挂起和恢复现场的花费可能更大，这样得不偿失。

​	因此，为了避免线程的切换而造成过多的开销，可以采用让当前线程进行**自旋**的操作，如果在自旋完成后前面锁定同步资源的线程已经释放了锁，那么当前线程就可以不必阻塞而是直接获取同步资源，从而避免切换线程的开销。这就是自旋锁。

##### 2.2 自旋锁的缺点

1. 不能代替阻塞。
2. 虽然避免了线程切换的开销，但是占用处理器的时间。
   - 锁占用时间越短，自旋等待的效果就会越好。
   - 锁占用时间越长，自旋的线程就只会白白的浪费处理器资源。
3. 解决一直白白占用处理器的情况，我们需要让自旋等待的时间具有一定的额度，如果自旋超过了限定次数则代表没有成功获得锁，应当挂起该线程。
4. JDK 1.4 使用 UseSpinning 来开启、JDK 1.6 默认开启，并引入了自适应的自旋锁(适应性自旋锁)
   - 自适应意味着自旋的时间（次数）不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。
   - 同一个锁对象上，自旋等待刚刚成功获得过锁，虚拟机会认为它还会成功，进而它将允许自旋等待持续相对更长的时间；但是对于某个锁，自旋很少成功获得过锁，那么就可能会直接略过它。

#### 3.无锁 VS 偏向锁 VS 轻量级锁 VS 重量级锁

1. 这个分类是用来甄别：多线程竞争同步资源的流程细节

2. 无锁：

   - 定义：不锁住资源，多个线程中只有一个修改资源成功，其它线程会重试。
   - 特点：修改操作在循环内进行，线程会不断的尝试修改共享资源，没有冲突就修改成功然后退出，否则就继续循环尝试，直到修改成功。

3. 偏向锁：

   - 定义：同一个线程执行同步资源时，自动获取资源。
   - 特点：在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径以提高性能，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只在置换 ThreadID 的时候依赖一次 CAS 指令。
   - 目的：通过对比 MarkWord 来解决加锁问题，避免了 CAS 操作

4. 轻量级锁：

   - 定义：多个线程竞争同步资源时，没有获得资源的线程通过自旋来等待锁释放。
   - 特点：
     - 偏向锁被其它线程访问时，会自动升级为轻量级锁。
     - 其它线程是通过**自旋**的形式尝试获取锁，**不会阻塞**，从而提高性能。
   - 目的：通过 CAS 操作和自旋来解决加锁问题，避免线程的堵塞和唤醒进而影响性能。

5. 重量级锁：

   - 定义：多个线程竞争同步资源时，其它所有等待锁的线程都会进入**阻塞状态**。
   - 特点：多个线程竞争时，若只有一个等待线程，当这个线程的自旋超过一定次数时，或者第三个线程来访时，那么轻量级锁会自动升级为重量级锁。

6. 总结：

   ​	无锁（无竞争，直接访问）→ 偏向锁（单线程重复访问，无需同步）→ 轻量级锁（多线程交替竞争，CAS自旋）→ 重量级锁（高竞争，线程阻塞，OS互斥量）。

#### 4.公平锁 VS 非公平锁

1. 公平锁：
   - 定义：多个线程按照申请锁的顺序来获得锁，线程进入队列中排队，FIFO获取锁。
   - 特点：
     - 优点：可以使得等待的线程不会饿死。
     - 缺点：整体吞吐率会比非公平锁的吞吐率低，同时除第一个线程以外所有线程都会阻塞，CPU唤醒阻塞线程的开销比非公平锁大。
2. 非公平锁：
   - 定义：多个线程加锁时尝试直接获取锁，获取不到的线程会去队列中排队等待。如果此时锁刚好可用，那么这个线程就可以无需阻塞直接获取锁。
   - 特点：
     - 优点：不必唤醒所有的线程，可以减少唤起线程的开销，整体的吞吐率提高。
     - 缺点：处于等待中的线程可能会饿死，或者很久才能获取锁。

#### 5.可重入锁 VS 非可重入锁

1. 可重入锁：
   - 定义：同一个线程在外部方法中访问并获得锁，再进入内部方法中会自动获取锁，不会因为之前获取过的锁没有释放而阻塞。
   - 特点：可以一定程度避免出现死锁的情况。

#### 6.独享锁 VS 共享锁

1. 独享锁：独享锁也叫排他锁，是指该锁一次只能被一个线程所持有。如果线程T对数据A加上排它锁后，则其他线程不能再对A加任何类型的锁。获得排它锁的线程即能读数据又能修改数据。
2. 共享锁：是指该锁可被多个线程所持有。如果线程T对数据A加上共享锁后，则其他线程只能对A再加共享锁，不能加排它锁。获得共享锁的线程只能读数据，不能修改数据。
3. 独享锁和共享锁通过 AQS 来实现的。

#### 7.死锁

- 定义：死锁是指两个或多个线程在执行过程中，因为争夺资源而造成的一种互相等待的状态。如果没有外部干预，这些线程将永远无法继续执行，程序将不可能正常终止。

- 四个必要条件：

  - **互斥条件**：该资源任意一个时刻只由一个线程占用。
  - **持有并等待条件**：一个线程因请求资源而阻塞时，对已获得的资源保持不放。
  - **不可剥夺条件**：线程持有的资源在其使用完之前，​**不能被其他线程强行抢占**，只能由线程主动释放。
  - **环路等待条件**：多个线程互相等待对方持有的资源，形成**环形依赖链**。例如：
    - 线程A持有资源1，等待资源2；
    - 线程B持有资源2，等待资源1。

- 预防和避免线程死锁：

  1. 预防死锁问题： 破坏死锁的产生的必要条件即可：

     1. **破坏请求与保持条件**：一次性申请所有的资源。
     2. **破坏不剥夺条件**：占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。
     3. **破坏循环等待条件**：靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放。破坏循环等待条件。

  2. 避免线程死锁：在资源分配时，借助于算法(如银行家算法)对资源分配进行计算评估，使其进入安全状态

     - **银行家算法核心总结**

       **目标**：在分配资源前预判安全性，避免死锁。

       **工作原理**

       1. **安全性预判**
          - 分配资源前，模拟进程执行并检查系统是否能满足所有进程的最大资源需求。
          - 若分配后存在至少一个**安全序列**（所有进程可依次完成），则允许分配；否则进程等待。
       2. **资源分配逻辑**
          - **检查条件**：当前剩余资源 ≥ 某进程的剩余需求（`Max - Allocated`）。
          - 安全序列构建：
            - 满足条件的进程加入安全序列，并**回收其已占用的资源**。
            - 重复检查，直到所有进程加入序列（安全状态）或无法满足（不安全状态）。
       3. **关键结论**
          - 存在安全序列 → 系统**一定不会死锁**。
          - 无死锁 ≠ 一定存在安全序列（安全状态是更严格的保证）。

### volatile 关键字

#### volatile 的作用是什么？

- 使用 `volatile` 一般用于保证变量的可见性和有序性。
  1. 可见性：绕过本地缓存，强制线程从主存中读取最新值，修改后立即返回主存。
  2. 有序性：通过插入内存屏障，禁止指令重排序。

#### volatile 如何保证可见性和有序性的？底层原理是什么？

- 可见性：通过 **MESI缓存一致性协议** 或 **总线锁** 强制同步主内存和线程工作内存。
- 有序性：通过 内存保障 实现
  - 写保障：确保 `volatile` 写操作前的指令不会重排序到 写之后。
  - 读保障：确保 `volatile` 读操作后的指令不会重排序到 读之前。
- :dango:内存保障的原因：由于现代 CPU 的 **多级缓存架构** 和 **指令重新排序优化** (对单线程无影响)，可能导致：
  - 不可见性：线程A修改了变量，但是线程B仍然读取了旧值。
  - 乱序执行：代码的编写顺序和代码的执行顺序可能不一致。

#### 它可以保证原子性/线程安全吗？

1. 不能保证原子性，而因为不能保证原子性进而不可以保证线程安全。

2. volatile 关键字是用来修饰变量，是为了保证数据之间的可见性，但是 `volatile` 仅保证单次 读/写 的原子性，不能解决多线程并发下的复合操作问题。比如 `i++` 这样的复合操作，包含了 读值，计算，写入三个步骤。

3. 三种保证原子性的方法：

   - 使用 `synchronized` 改进：

     ```java
     public synchronized void increase() {
         inc++;
     }
     ```

     使用 `AtomicInteger` 改进：

     ```java
     public AtomicInteger inc = new AtomicInteger();
     
     public void increase() {
         inc.getAndIncrement();
     }
     ```

     使用 `ReentrantLock` 改进：

     ```java
     Lock lock = new ReentrantLock();
     public void increase() {
         lock.lock();
         try {
             inc++;
         } finally {
             lock.unlock();
         }
     }
     ```

### synchronized 关键字

#### 它是什么，有什么作用？

1. `synchronized` 是 Java 提供的 **内置锁机制**，用于解决多线程环境下的 **线程安全问题**。
2. 它的核心作用是：
   - **原子性**：确保同一时间只有一个线程能执行同步代码块/方法。
   - **可见性**：线程释放锁前，会将修改的变量刷回主内存，其他线程能立即看到最新值。
   - **有序性**：同步块内的代码不会被指令重排序优化打乱执行顺序。

#### 如何使用 synchronized？

​	`synchronized` 关键字的使用方式主要有下面 3 种：

1. 修饰实例方法（锁的是当前 **对象实例** ）
2. 修饰静态方法（锁的是当前 **类** ）
3. 修饰代码块 （锁的是当前 对象/类）

#### 构造方法可以用 synchronized 修饰么？

1. 构造方法不能使用 synchronized 关键字修饰。
2. 构造方法本身就是线程安全的（每次调用构造方法都会创建新对象，不存在多线程竞争的问题），加上`synchronized`毫无意义而且会导致语法错误。

#### synchronized 底层原理了解吗？

1. #### **核心实现根基：Java对象头与Monitor机制**

   1. **Java对象头（存储锁状态）**
      - Mark Word（核心字段) :
        - 存储对象 hashCode、分代年龄、**锁标志位**（锁状态标识）。
        - 采用动态数据结构，根据锁状态复用存储空间（如偏向锁时存储线程ID）。
      - **Klass Pointer**：指向类元数据，与锁无关。
   2. **Monitor（监视器锁）**
      - 每个Java对象隐式关联一个Monitor，线程通过竞争Monitor的 ownership 实现同步。
      - Monitor 关键结构：
        - `Owner`：记录当前持有锁的线程。
        - `EntryList`：阻塞状态，等待锁的线程队列。
        - `WaitSet`：调用 `wait()` 后的线程被阻塞，被放入这个队列。

2. #### **锁的获取与释放流程**

   1. 当多个线程同时访问一同步代码时：
      - 先进入 `EntryList` 集合，进行阻塞等待，
      - 成功获取锁后，Monitor的 `Owner` 字段设置为当前线程，同时计数器 `Count` +1。
   2. 线程调用同步对象的方法 wait()方法：
      - 释放当前持有的锁，将 `Owner` 置空，同时计数器 `Count` -1，同时线程进入`WaitSet`中等待唤醒。
      - 当线程执行完同步代码后，也将 `owner` 和 `count` 重置。

3. #### **锁升级优化（JDK 6+）**

   为提高性能，`synchronized` 采用 **自适应锁状态切换**：

   |    锁状态    |         适用场景         |            实现方式            | 开销 |
   | :----------: | :----------------------: | :----------------------------: | :--: |
   |   **无锁**   |       新创建的对象       |               -                |  无  |
   |  **偏向锁**  |      单线程重复访问      |      Mark Word记录线程ID       | 极低 |
   | **轻量级锁** | 多线程交替执行（低竞争） |            CAS自旋             |  低  |
   | **重量级锁** |        高竞争场景        | 操作系统Mutex Lock（线程阻塞） |  高  |

   **锁升级特点**：

   - 单向升级（无锁 → 偏向 → 轻量级 → 重量级），不可降级。
   - **JDK 15后偏向锁被废弃**：因高并发场景下维护成本高，收益有限。

#### JDK1.6之后的 synchronized 底层做了哪些优化？

​	在 Java 6 之后， `synchronized` 引入了大量的优化如自旋锁、适应性自旋锁、锁消除、锁粗化、锁升级等技术来减少锁操作的开销。

#### synchronized 的偏向锁为什么被废弃了？

​	在 JDK 15 中，偏向锁被标记为废弃（默认禁用），JDK 18中废除了，原因包括：

1. **维护成本高**：偏向锁的撤销流程复杂（尤其是存在大量锁竞争时）。
2. **收益下降**：现代应用多为高并发场景，偏向锁（适合单线程）的实际收益有限。
3. **性能权衡**：直接使用轻量级锁（自旋）或重量级锁更简单高效。

#### synchronized 和 volatile 有什么区别？

|   **维度**   |    `synchronized`     |     `volatile`      |
| :----------: | :-------------------: | :-----------------: |
| **作用范围** |      代码块/方法      |        变量         |
|  **原子性**  |     ✔️（复合操作）     |  ❌（仅单次读/写）   |
|  **可见性**  | ✔️（锁释放前刷主内存） | ✔️（直接读写主内存） |
|  **有序性**  |   ✔️（锁内串行执行）   |   ✔️（禁止重排序）   |
|   **阻塞**   |     ✔️（线程阻塞）     |     ❌（无阻塞）     |
|   **性能**   |        高开销         |       低开销        |

### ReentrantLock

#### ReentrantLock是什么？

​	`ReentrantLock` 是JUC中的一个可重入互斥锁，它提供了与 `Synchronized` 关键字类似的功能，但是更加的灵活且强大。

​	其主要特点有：

- 可重入性：同一个线程可以多次获得同一把锁(避免死锁)。
- 灵活性：可以选择公平锁/非公平锁、可中断锁、可超时锁、变量条件等高级功能。
- 手动获取/释放：需要显示的用 `lock` 和 `unlock` 来获取/释放 锁。

#### ReentrantLock的实现原理？

​	 `ReentrantLock` 里有一个内部类 `Sync`，`Sync` 继承自AQS，添加锁和释放锁的大多数操作都是在 `Sync` 中实现的，所以可以说底层原理是基于 AQS。

#### 它和公平锁的关系

​	`ReentrantLock` 默认使用 非公平锁，也可以通过传入一个 `true` 值来显示的指定使用公平锁。

#### 公平锁和非公平锁的区别

- 公平锁：想要获取锁的线程会依次排队，使用的是FIFO的队列结构，不会出现线程饿死的情况，但是由于线程经常性的切换，会导致性能不好。
- 非公平锁：想要获取锁的线程会先去插个队，看看自己能不能直接获得锁，能的话，就直接操作并修改共享资源，省去线程切换的开销；不能的话就会到后边乖乖排队。因此性能会更好，但是可能有线程长时间不能获得锁，或者是饿死的情况发生。

#### synchronized 和 ReentrantLock 有什么区别？

|     特性     |      `ReentrantLock`       |     `synchronized`     |
| :----------: | :------------------------: | :--------------------: |
|  **可重入**  |           ✅ 支持           |         ✅ 支持         |
|  **公平性**  |  ✅ 可配置公平锁或非公平锁  |        ❌ 非公平        |
|  **可中断**  |  ✅ `lockInterruptibly()`   |        ❌ 不支持        |
| **超时获取** | ✅ `tryLock(timeout, unit)` |        ❌ 不支持        |
| **条件变量** |  ✅ 可绑定多个 `Condition`  | ❌ 仅 `wait()/notify()` |
|   **性能**   |      高竞争场景下更优      |   JDK 6+ 优化后接近    |

### ThreadLocal

#### ThreadLocal有什么用？

​	`ThreadLocal` 主要是为了实现线程间的 **数据隔离** 并减少 **同步开销**，它是 `java.lang` 包中的一个类，可以为每个线程提供独立的变量副本，并且使得每个线程可以使用`get()` 或`set()` 来修改自己的副本而不影响其它的线程。

#### 它的原理是什么？

​	`ThreadLocal` 能够实现数据隔离，主要依赖于`Thread` 实例内部的 `ThreadLocalMap` 成员变量，`ThreadLocalMap` 主要是用于存储线程中所有的 `ThreadLocal` 变量及所对应的值。

#### ThreadLocal的生命周期管理？

​	在使用 `ThreadLocal` 过程中，要认真管理它的生命周期，尤其是在线程池复用线程的场景下，可能会导致`ThreadLocal` 变量可能保存的还是旧的数据，这样会导致 **内存泄露** 或者 **数据错乱** 。 

- 内存泄漏：
  - 原因：归咎于 `ThreadLocalMap` 中的 key 和 value 的引用机制。
    - key 是弱引用：key 存储的是 ThreadLocal 的实例，是 ThreadLocal 的弱引用，当 ThreadLocal 实例不再被任何强引用所指向时，垃圾回收器在下次 GC 时会回收该实例，导致 ThreadLocal 中对应的 key 变为 null。
    - 而 value 是强引用：value 就是我们需要存储的值，它是强引用，即使 key 被GC回收了，value 还是被 ThreadLocalMap.Entry 强引用而存在，无法被GC回收。
    - 总结：当 ThreadLocal 实例不再被任何强引用指向时，key 因为是 ThreadLocal 的弱引用会被GC回收，但是 ThreaLocalMap.Entry 的强引用了 value，所以当线程一直存活的话，ThreadLocalMap 也会一直存在，进而导致 value 不会被GC回收，就造成了内存泄露。
  - 解决方案：
    - 在使用 ThreadLocal 后，使用 `remove()` 回收对应的 Entry，彻底解除内存泄露的风险。
    - 在线程池等线程复用的情况下，使用 try-finally 块确保即使发生了异常，也会一定执行 `remove()` 

#### 如何跨线程传递ThreadLocal的值？

1. 使用 `InheritableThreadLocal`：在创建子线程时，子线程会自动继承父线程中的 ThreadLocal 的值。但是它无法支持线程池场景下的 ThreadLocal 值传递。
2. 使用 `TransmittableThreadLocal`：TTL是阿里巴巴开源的工具类，继承并加强了 `InheritableThreadLocal` 类，使得其可以在线程池场景下支持 `ThreadLocal` 的值传递。

### 线程池

#### 什么是线程池？

​	线程池就是管理一些列线程的资源池。有任务要处理时，就直接从线程池中去拿线程来用，处理完后线程不会销毁，而是等待获取新的任务。

#### 为什么要用线程池？

​	线程池也是一种池化技术，都是为了减少每次获取资源的消耗，提高对资源的利用率。

1. 降低资源消耗。通过重复利用已经创建好的线程，避免了一直创建或销毁线程的开销。
2. 提高响应速度。任务到达时，我们不需要再创建一个线程来执行任务，直接拿就好。
3. 提高线程的可管理性。线程是稀缺资源，使用线程池可以统一分配、调优和监控线程。
4. 防止资源耗尽。通过限制线程的数量，可以避免因为创建了过多的线程而导致程序崩溃。

#### 如何创建线程池？

1. 可以使用 `ThreadPoolExecutor` 构造函数来创建线程池。
2. 使用 `Executor` 框架中工具类 `Executors` 来创建线程池，有四种预制的类型：
   1. `FixedThreadPool`：固定线程数量的线程池。当有一个新任务提交时，如果线程池中有空闲线程，就直接去执行任务；若没有，新任务会被暂存到一个任务队列中，等有空闲线程了，就去处理任务队列中的任务。(可以选择线程数量，使用了阻塞队列)
   2. `SingleThreadExecutor`：只有一个线程的线程池。除第一个任务以外，所有的任务都会被放置在任务队列中，等待线程完成第一个任务。（只有一个线程，使用了阻塞队列）
   3. `CachedThreadPool`：可以根据实际情况调整线程数量的线程池。当一个新任务提交时，如果线程池中有空闲线程可以复用，那么就优先使用空闲线程来执行任务；若没有，则创建一个新线程来处理任务。（线程会实时变化，不使用队列）
   4. `ScheduleThreadPool`：可以延时处理任务或定期处理任务的线程池。（使用了延迟阻塞队列）

#### 为什么不推荐使用内置线程池？

​	不推荐使用 `Executors` 提供的内置线程池的主要原因：

1. 任务队列的长度都是 `Integer.MAX_VALUE` ，那么就会面临大量任务请求堆积的问题，从而导致OOM
2. 缺乏灵活性，无法自定义参数。

#### 线程池常见的参数？如何解释？

1. `int corePoolSize`：即使空闲，也要保留在线程池中的线程数，除非设置 `allowCoreThreadTimeOut`，不能小于0。
2. `int maximumPoolSize`：池中允许的最大线程数，不能小于1或小于核心线程数
3. `long keepAliveTime`：当池中线程数超过了核心线程数时，池中多余空闲线程在终止前等待新任务的最长时间，超过这个时间，线程就会被回收，不能小于0。
4. `TimeUnit unit`：`keepAliveTime` 的时间单位。
5. `BlockingQueue<Runnable> workQueue` ：执行任务之前用于容纳任务的队列。
6. `ThreadFactory threadFactory`：执行器创建新线程处理程序时使用的工厂。
7. `RejectedExecutionHandler handler`：饱和拒绝策略：由于达到了线程边界和任务队列边界任务，从而导致执行受阻时使用的处理程序。`ThreadPoolExecutor` 有以下四个拒绝策略：
   - `AbortPolicy`，中止策略：抛出 `RejectedExecutionException` 来拒绝处理新的任务。
   - `CallerRunsPolicy` ，调用者运行策略：由调用者线程来执行该任务，如果执行程序关闭，就会丢弃这个任务。适合每个任务都很重要，不能丢弃、调用者线程能够接收额外负载或延迟的情况。
   - `DiscardPolicy` ，丢弃策略：直接丢弃任务，不做任何处理。
   - `DiscardOldestPolicy` ，丢弃最老策略：将丢弃队列中最老的任务，然后尝试重写提交新任务。

#### 线程池中核心线程会被回收吗？

​	默认不会回收，即使空闲了也会一直等待，但是可以设置使用 allowCoreThreadTimeOut 来设置核心线程存活时间，这样就会收回空闲核心线程了。

#### 核心线程空闲时处于什么状态？

- 没有设置 allowCoreThreadTimeOut ，就会一直处于 `WAITING` 状态，等待获取任务。
- 设置了 allowCoreThreadTimeOut，如果阻塞等待时间超过了核心线程最长存活时间，那么这个线程就会退出工作，并且从线程池中移除，状态为 `TERMINATED`

#### 线程池的拒绝策略有什么？

AbortPolicy、CallerExecutionPolicy、DiscardPolicy、DiscardOldestPolicy。

#### 如果不允许丢弃任务，应该选择哪个拒绝策略？

使用 CallerExecutionPolicy：调用者执行策略

#### CallerRunsPolicy 拒绝策略有什么风险呢？如何解决？

1. 风险：如果调用了这个策略的任务耗时会很长，并且是由主线程所提交的任务，那么就有可能导致主线程阻塞，影响程序的正常运行。
2. 解决方案：
   - 增加阻塞队列`BlockingQueue`的大小并调整堆内存以容纳更多的任务，确保任务能够被准确执行。
   - 调整线程池的`maximumPoolSize` （最大线程数）参数，这样可以提高任务处理速度，避免累计在 `BlockingQueue`的任务过多导致内存用完。
   - 任务持久化：
     - 可以设计一张任务表将任务存储到 MySQL 数据库中。
     - 可以Redis 缓存任务。
     - 可以将任务提交到消息队列中。

#### 线程池常见的阻塞队列？

|       队列类型        |       容量       |         适用场景          |                 线程池示例                  |
| :-------------------: | :--------------: | :-----------------------: | :-----------------------------------------: |
| `LinkedBlockingQueue` | 可选（默认无界） | 固定大小线程池（需防OOM） | `FixedThreadPool` <br>`SingleThreadExcutor` |
|  `SynchronousQueue`   |        0         |     高吞吐量、短任务      |             `CachedThreadPool`              |
|  `DelayedWorkQueue`   |  无界（堆结构）  |     定时任务/延迟任务     |        `ScheduledThreadPoolExecutor`        |
| `ArrayBlockingQueue`  |       固定       | 需严格控制队列大小的场景  |                自定义线程池                 |

#### 线程池处理任务的流程了解吗？

1. 如果线程池状态不是运行中，比如说是SHUTDOWN、STOP或其它状态，则拒绝执行任务
2. 当前线程数 < corePoolSize，创建新线程执行任务。
3. 当前线程数 ≥ corePoolSize，将任务放入任务队列。
4. 任务队列已满, 当前线程数 < maximumPoolSize，创建新的非核心线程来执行任务。
5. 任务队列已满, 当前线程数 > maximumPoolSize，执行饱和拒绝策略。

#### 线程池中线程异常后，销毁还是复用？

- 使用`execute()`时，未捕获异常导致线程终止，线程池创建新线程替代；
- 使用`submit()`时，异常被封装在`Future`中，线程继续复用。

### AQS框架

#### AQS是什么？

​	AQS（AbstractQueuedSynchronizer）是JUC锁中的一个核心框架，用于构建锁和其它同步器的基础类。提供了一种模板方法设计模式，可以让我们通过继承AQS并实现其提供的抽象方法，来构建我们自己的同步器：独占锁、共享锁、信号量等。

#### AQS的原理是什么？

​	AQS 的核心原理是基于一个原子整型变量 `state` 来进行 **状态管理** 和 一个 **FIFO等待队列** 来完成获取资源的线程的排队。

**状态管理**

- **state 变量**：AQS 使用一个 `volatile int state` 来表示同步状态。例如，在独占锁中，`state=0` 表示锁空闲，`state=1` 表示锁被占用。
- **状态操作**：提供了 `getState()`、`setState()` 和 `compareAndSetState()` 等方法来操作和更新同步状态，确保线程安全。

**FIFO 等待队列**

- **CLH 队列变种**：AQS 内部维护一个基于 CLH 队列变种的双向链表队列，用于存放等待获取锁的线程。
- **节点结构**：每个节点代表一个等待线程，包含线程引用、等待状态等信息。
- **入队和出队**：当线程尝试获取锁失败时，会被封装成一个节点并加入到队列尾部；当锁被释放时，AQS 会从队列头部唤醒下一个等待的线程。

#### AQS 的几种常见应用场景？

| 同步工具               | 同步工具与AQS的关联                                          |
| :--------------------- | :----------------------------------------------------------- |
| ReentrantLock          | 使用AQS保存锁重复持有的次数。当一个线程获取锁时，ReentrantLock记录当前获得锁的线程标识，用于检测是否重复获取，以及错误线程试图解锁操作时异常情况的处理。 |
| Semaphore              | 使用AQS同步状态来保存信号量的当前计数。tryRelease会增加计数，acquireShared会减少计数。 |
| CountDownLatch         | 使用AQS同步状态来表示计数。计数为0时，所有的Acquire操作（CountDownLatch的await方法）才可以通过。 |
| ReentrantReadWriteLock | 使用AQS同步状态中的16位保存写锁持有的次数，剩下的16位用于保存读锁的持有次数。 |
| ThreadPoolExecutor     | Worker利用AQS同步状态实现对独占线程变量的设置（tryAcquire和tryRelease）。 |

#### Semaphore

- **描述**：`Semaphore` 用于控制同时访问特定资源的线程数量，通过许可证机制实现。
- **实现**：基于 AQS 的共享模式，`state` 表示可用许可证的数量。
- **应用**：限制数据库连接池的并发访问数、控制任务提交速率等。

#### CountDownLatch

- **描述**：`CountDownLatch` 允许一个或多个线程等待其他线程完成操作。
- **实现**：基于 AQS 的共享模式，初始 `state` 设置为计数器的值，每个 `countDown()` 调用减少 `state`，当 `state=0` 时，所有等待线程被唤醒。
- **应用**：主线程等待多个子线程完成任务后再继续执行。

#### CyclicBarrier

- **描述**：`CyclicBarrier` 让一组线程互相等待，直到所有线程都到达屏障点，然后一起继续执行。
- **实现**：虽然 `CyclicBarrier` 不直接继承 AQS，但其内部使用了类似 AQS 的机制来管理线程的等待和唤醒。
- **应用**：多线程计算中，各线程完成各自部分后，需要汇总结果时使用。