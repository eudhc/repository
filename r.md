
# 操作系统课程设计实验报告


# <a id="_Toc122817971"></a>1\.准备

## <a id="_Toc122817972"></a>1\.1 Ubantu

在VMware Workstation中安装Ubuntu 18\.04LTS虚拟机，并进行基本的配置。

##  <a id="_Toc122817973"></a>1\.2 Git

1．下载安装好git之后，使用ssh公钥与gitlab进行连接。

2．使用git clone将三字母仓库克隆到自己的虚拟机上。

3．测试git push命令是否能够正常使用。

## <a id="_Toc122817974"></a>1\.3 Pitnos

1．安装 qemu

sudo apt\-get install qemu

2．从 Git 公共库获取最新 Pintos

git clone git://pintos\-os\.org/pintos\-anon

3．进入pintos/src/utils/pintos\-gdb 用 VIM 打开，编辑 GDBMACROS 变量，将Pintos  完整路径赋给该变量。

4．VIM 打开 Makefile 并将 LOADLIBES 变量名编辑为 LDLIBS

5．在/src/utils 中输入 make 来编译 utils

6．编辑/src/threads/Make\.vars（第 7 行）：更改 bochs 为 qemu

7．在/src/threads 并运行来编译线程目录 make

8．编辑/utils/pintos（第 103 行）：替换 bochs 为 qemu

编辑/utils/pintos（257 行）：替换 kernel\.bin 为完整路径的 kernel\.bin

编辑/utils/pintos（621 行）：替换 qemu 为 qemu\-system\-x86\_64

9\. 编辑/utils/Pintos\.pm（362 行）：替换 loader\.bin 为完整路径的 loader\.bin

10\.~/\.bashrc 并添加 export PATH=/home/…/pintos/src/utils:$PATH 到最后一行。

11．重新打开终端输入 source ~/\.bashrc 并运行

12．在 Pintos 下打开终端输入 pintos run alarm\-multiple

# <a id="_Toc122817975"></a>2\.源码分析

## <a id="_Toc122817976"></a>2\.1 背景知识

第一步是阅读和理解初始线程系统的代码。Pintos已经实现了线程创建和线程结束、一个在线程之间切换的简单调度程序、以及同步原语（信号量、锁、条件变量和优化屏障）。

当线程创建后，您需要创建一个新的上下文用于调度。您提供一个将在此上下文中运行的函数，作为thread\_create\(\)的参数。线程第一次被调度并运行时，它从该函数的开头开始并在该上下文中执行。当函数返回时，线程终止。因此，每个线程的行为就像在Pintos中运行的小程序一样，传递给thread\_create\(\)的函数的行为就像main\(\)一样。

在任意一个给定的时刻，只有一个线程在运行，其余线程（如果有的话）变为非活动状态。调度程序决定下一步运行哪个线程。（如果在某个给定时刻没有线程是就绪的，则运行在idle\(\)中实现的特殊“idle”线程。）当一个线程需要等待另一个线程执行某个操作时，同步原语可以强制上下文切换\.

上下文切换的机制位于“threads/switch\.S”中，这是80x86的汇编代码。它可以保存当前正在运行的线程的状态并恢复我们要切换的线程的状态。

## <a id="_Toc122817977"></a>2\.2 源文件

（1）文件夹功能说明

文件夹     | 功能
-------- | -----
threads  | 基本内核的代码
devices  | IO 设备接口，定时器、键盘、磁盘等代码
lib  | 实现了 C 标准库，该目录代码会与 Pintos kernel 一起编译，用户的程序也要在此目 录下运行。内核程序和用户程序都可以使用 \#include 的方式来引入这个目录下的头 文件

（2）threads/ 文件夹中文件说明
文件     | 功能
-------- | -----
loader\.h  | 内核加载器
loader\.S  | 
kernel\.lds\.S  | 连接脚本，用于连接内核，设置负载地址的内核
init\.h  | 内核的初始化，包括 main\(\) 函数
init\.c  | 
thread\.h  | 实现基础线程功能
thread\.c  | 
switch\.h  | 汇编语言实现常规的用于交换的线程
switch\.S  | 
start\.S  | 跳转到主函数

（3）devices/ 文件夹中文件说明

文件  | 功能
-------- | -----
timer\.h  | 实现系统计时器，默认使每秒运行 100 次
timer\.c  | 
vga\.h  | 显示驱动程序，负责将文本打印在屏幕上
vga\.c  | 
serial\.h  | 串行端口驱动程序，通过 printf\(\) 函数调用，将内容并传入输入层
serial\.c  | 
disk\.h  | 支持磁盘进行读写操作
disk\.c  | 
kbd\.h  | 支持磁盘进行读写操作
kbd\.c  | 
input\.h  | 输入层程序，将传入的字符组成输入队列
input\.c  | 
intq\.h  | 中断队列程序，管理内核线程和中断处理程序
intq\.c  | 

## <a id="_Toc122817978"></a>2\.3 项目分析

1.进程管理这一项目的入手点是timer_sleep()函数，以下展示该函数的整体结构：

    /\* Sleeps for approximately TICKS timer ticks\.  Interrupts must

       be turned on\. \*/

    void timer\_sleep\(int64\_t ticks\)

    \{

      int64\_t start = timer\_ticks\(\); //获取开始的时间

      ASSERT\(intr\_get\_level\(\) == INTR\_ON\);

      while \(timer\_elapsed\(start\) < ticks\) //查看当前时间是否小于设定的睡眠时间

      thread\_yield\(\);                    //将当前线程放入就绪队列，并调度下一个线程

    \}

在第5行中，首先获得了线程休眠的开始时间，timer\_ticks\(\)函数在获取时间的过程中采用了关中断保存程序状态字，而后开中断恢复程序状态字的办法以防止在执行过程中发生中断，由于后续程序也用到了开关中断的操作，因此将在接下来进行介绍。

在第7行中，断言了当前中断的状态，确保中断是打开的状态。

2.接下来是重点部分，首先看timer_elapsed()函数，其整体结构如下：

    /\* Returns the number of timer ticks elapsed since THEN, which

      should be a value once returned by timer\_ticks\(\)\. \*/

    int64\_t

    timer\_elapsed\(int64\_t then\)

    \{

      return timer\_ticks\(\) \- then;

    \}

我们可以看到这个函数实际是计算当前线程已经休眠的时间，它将结果返回至timer_sleep()函数后，利用while循环判断休眠时间是否已经达到ticks时间（这里的ticks时间是传入的拟休眠时间的局部变量，而不是全局变量系统启动后到现在的时间），如果没有达到，就将不停的进行thread_yield()。

3\.thread\_yield\(\)函数的整体结构如下所示：

    /\* Yields the CPU\.  The current thread is not put to sleep and

       may be scheduled again immediately at the scheduler's whim\. \*/

    void thread\_yield\(void\)

    \{

      struct thread \*cur = thread\_current\(\); //获取当前页面的初始位置（指针指向开始）

      enum intr\_level old\_level;

      ASSERT\(\!intr\_context\(\)\);

      old\_level = intr\_disable\(\);                //关中断

      if \(cur \!= idle\_thread\)                    //如果当前线程不是空闲线程

        list\_push\_back\(&ready\_list, &cur\->elem\); //把当前线程放入就绪队列

      cur\->status = THREAD\_READY;                //修改程序状态为就绪

      schedule\(\);                                //调度下一个线程

      intr\_set\_level\(old\_level\);                 //开中断

    \}

（1）页面指针的获取

在第5行，cur指针通过调用thread_current()函数来获取指向页面初始位置的指针，由于该函数也进行了多级嵌套，在此仅简要描述一下函数功能实现的流程。首先，这一函数获取了esp寄存器的值，这一寄存器是指向栈顶的寄存器，为了获取指向页首的指针，我们知道Pintos中一页的大小是2的12次方，因此其做法就是将1这个数字在二进制下左移12位并取反后，再与esp寄存器中的值相与，即可获得页首指针。

（2）原子化操作

所谓原子化操作即为开篇所提到的关中断和开中断操作，其分别由以下两个语句实现：

    old_level = intr_disable();                //关中断

其他操作

    intr_set_level(old_level);                 //开中断

其基本实现步骤是利用堆栈的push和pop语句得到程序状态字寄存器的值，并利用CLI指令关中断，恢复时，将值mov进寄存器，并STI开中断。

（3）线程的切换

这一步骤体现在11-14行代码中，如果当前线程不是空闲线程，则把它加入就绪队列，加入的方式是通过指针的改变使其与前后的线程关联起来，形成一个队列，并将这一线程修改为就绪状态。

由此可以得知thread_yield()函数的作用便是将当前线程放入就绪队列，并调度下一线程。而timer_sleep()函数便是在限定的时间内，使运行的程序不停的放入就绪队列，从而达到休眠的目的。

当然这样去做的一大缺点，就是线程不断的在运行和就绪的状态来回切换，极大的消耗资源，由此我们将进行改进。

# <a id="_Toc122817979"></a>3.源码实现

## <a id="_Toc122817980"></a>3.1 timer_sleep()函数的重新实现

### <a id="_Toc122817981"></a>3.1.1 实现思路

由于原本的timer_sleep()函数采用运行和就绪间切换的方法过于消耗CPU的资源，考虑到Pintos提供了线程阻塞这一模式（见下方线程状态结构体），因此我打算在线程结构体中加入一个用于记录线程睡眠时间的变量，通过利用Pintos的时钟中断（见下方时间中断函数），即每个tick将会执行一次，这样每次检测时将记录线程睡眠时间的变量自减1，当该变量为0时即可代表能够唤醒该线程，从而避免资源的过多开销。

线程状态结构体：

    enum thread_status

    \{

      THREAD\_RUNNING,     /\* Running thread\. \*/

      THREAD\_READY,       /\* Not running but ready to run\. \*/

      THREAD\_BLOCKED,     /\* Waiting for an event to trigger\. 阻塞状态\*/

      THREAD\_DYING        /\* About to be destroyed\. \*/

    \};

时间中断函数：

    /\* Timer interrupt handler\. \*/

    static void

    timer\_interrupt\(struct intr\_frame \*args UNUSED\)

    \{

      ticks\+\+;

      thread\_tick\(\);

    \}

### <a id="_Toc122817982"></a>3\.1\.2 实现方法

（1）改写线程结构体，在结构体中增加记录线程睡眠时间的变量ticks\_blocked

    struct thread

      \{

        /\* Owned by thread\.c\. \*/

        tid\_t tid;                          /\* Thread identifier\. \*/

        enum thread\_status status;          /\* Thread state\. \*/

        char name\[16\];                      /\* Name \(for debugging purposes\)\. \*/

        uint8\_t \*stack;                     /\* Saved stack pointer\. \*/

        int priority;                       /\* Priority\. \*/

        struct list\_elem allelem;           /\* List element for all threads list\. \*/

        int64\_t ticks\_blocked;              //增加的变量\->记录要阻塞的时间

        /\* Shared between thread\.c and synch\.c\. \*/

        struct list\_elem elem;              /\* List element\. \*/

    \#ifdef USERPROG

        /\* Owned by userprog/process\.c\. \*/

        uint32\_t \*pagedir;                  /\* Page directory\. \*/

    \#endif

        /\* Owned by thread\.c\. \*/

        unsigned magic;                     /\* Detects stack overflow\. \*/

      \};

（2） 然后在线程被创建的时候初始化ticks\_blocked为0， 加在thread\_create函数内：

    t\->ticks\_blocked = 0;

（3） 然后修改时钟中断处理函数， 加入线程sleep时间的检测， 加在            timer\_interrupt内：

    thread\_foreach \(blocked\_thread\_check, NULL\);

（4） 这里的thread\_foreach就是对每个线程都执行blocked\_thread\_check这个函数：

    /\* Invoke function 'func' on all threads, passing along 'aux'\.

       This function must be called with interrupts off\. \*/

    void

    thread\_foreach \(thread\_action\_func \*func, void \*aux\)

    \{

      struct list\_elem \*e;

      ASSERT \(intr\_get\_level \(\) == INTR\_OFF\);

      for \(e = list\_begin \(&all\_list\); e \!= list\_end \(&all\_list\);

       e = list\_next \(e\)\)

        \{

      struct thread \*t = list\_entry \(e, struct thread, allelem\);

      func \(t, aux\);

    \}

    \}

aux就是传给这个函数的参数。

（5） 然后， 给thread添加一个方法blocked\_thread\_check即可：

thread\.h中声明：

    void blocked\_thread\_check \(struct thread \*t, void \*aux UNUSED\);

thread\.c中声明：

    /\* Check the blocked thread \*/

    void

    blocked\_thread\_check \(struct thread \*t, void \*aux UNUSED\)

    \{

      if \(t\->status == THREAD\_BLOCKED && t\->ticks\_blocked > 0\)

      \{

      t\->ticks\_blocked\-\-;

      if \(t\->ticks\_blocked == 0\)

      \{

          thread\_unblock\(t\);

      \}

      \}

    \}

thread\_unblock：

    void

    thread\_unblock \(struct thread \*t\)

    \{

      enum intr\_level old\_level;

      ASSERT \(is\_thread \(t\)\);

      old\_level = intr\_disable \(\);

      ASSERT \(t\->status == THREAD\_BLOCKED\);

      list\_push\_back \(&ready\_list, &t\->elem\);

      t\->status = THREAD\_READY;

      intr\_set\_level \(old\_level\);

    \}

至此timer\_sleep\(\)的唤醒机制便编写完成了。

### <a id="_Toc122817983"></a>3\.1\.3 实现结果

此时，在/threads/build重新make check的结果如下所示：

pass tests/threads/alarm-single

pass tests/threads/alarm-multple

pass tests/threads/alarm-simultaneous

pass tests/threads/alarm-zero

pass tests/threads/alarm-negative



## <a id="_Toc122817984"></a>3\.2 优先级调度的实现

### <a id="_Toc122817985"></a>3\.2\.1 保证插入线程至就绪队列时保持优先级队列

#### 3\.2\.1\.1 实现思路

这里实现优先级调度的核心思想就是： 维持就绪队列为一个优先级队列。换一种说法： 我们在插入线程到就绪队列的时候保证这个队列是一个优先级队列即可。

我们在下面三种情况会把线程丢进就绪队列中：

1\. thread\_unblock

2\. init\_thread

3\. thread\_yield

那么我们只要在扔的时候维持这个就绪队列是优先级队列即可。

由于Pintos预置函数list\_insert\_ordered\(\)的存在，可以直接使用这个函数实现线程插入时按照优先级完成，因此只需要将涉及直接在末尾插入线程的函数中的语句进行替换即可。

list\_insert\_ordered（）：

        /\* Inserts ELEM in the proper position in LIST, which must be

       sorted according to LESS given auxiliary data AUX\.

       Runs in O\(n\) average case in the number of elements in LIST\. \*/

    void

    list\_insert\_ordered \(struct list \*list, struct list\_elem \*elem,

                     list\_less\_func \*less, void \*aux\)

    \{

      struct list\_elem \*e;

      ASSERT \(list \!= NULL\);

      ASSERT \(elem \!= NULL\);

      ASSERT \(less \!= NULL\);

      for \(e = list\_begin \(list\); e \!= list\_end \(list\); e = list\_next \(e\)\)

        if \(less \(elem, e, aux\)\)

          break;

      return list\_insert \(e, elem\);

    \}

#### 3\.2\.1\.2 实现方法

实现一个优先级比较函数thread\_cmp\_priority\(\):

    /\* priority compare function\. \*/

    bool

    thread\_cmp\_priority \(const struct list\_elem \*a, const struct list\_elem \*b, void \*aux UNUSED\)

    \{

      return list\_entry\(a, struct thread, elem\)\->priority > list\_entry\(b, struct thread, elem\)\->priority;

    \}

调用Pintos预置函数list\_insert\_ordered\(\)替换thread\_unblock\(\)中的list\_push\_back\(\)函数：

    void

    thread\_unblock \(struct thread \*t\) 

    \{

      enum intr\_level old\_level;

      ASSERT \(is\_thread \(t\)\);

      old\_level = intr\_disable \(\);

      ASSERT \(t\->status == THREAD\_BLOCKED\);

      list\_insert\_ordered \(&ready\_list, &t\->elem, \(list\_less\_func \*\) &thread\_cmp\_priority, NULL\);

      t\->status = THREAD\_READY;

      intr\_set\_level \(old\_level\);

    \}

然后同理调用Pintos预置函数list\_insert\_ordered\(\)替换init\_thread\(\)、thread\_yield\(\)中的list\_push\_back\(\)函数：

    init\_thread\(\)：

    static void

    init\_thread \(struct thread \*t, const char \*name, int priority\)

    \{

      enum intr\_level old\_level;

      ASSERT \(t \!= NULL\);

      ASSERT \(PRI\_MIN <= priority && priority <= PRI\_MAX\);

      ASSERT \(name \!= NULL\);

      memset \(t, 0, sizeof \*t\);

      t\->status = THREAD\_BLOCKED;

      strlcpy \(t\->name, name, sizeof t\->name\);

      t\->stack = \(uint8\_t \*\) t \+ PGSIZE;

      t\->priority = priority;

      t\->magic = THREAD\_MAGIC;

      old\_level = intr\_disable \(\);

      list\_insert\_ordered \(&all\_list, &t\->allelem, \(list\_less\_func \*\) &thread\_cmp\_priority, NULL\);

      intr\_set\_level \(old\_level\);

    }

    thread\_yield\(\)：

    void

    thread\_yield \(void\) 

    \{

      struct thread \*cur = thread\_current \(\);

      enum intr\_level old\_level;

  

      ASSERT \(\!intr\_context \(\)\);

      old\_level = intr\_disable \(\);

      if \(cur \!= idle\_thread\) 

        list\_insert\_ordered \(&ready\_list, &cur\->elem, \(list\_less\_func \*\) &thread\_cmp\_priority, NULL\);

      cur\->status = THREAD\_READY;

      schedule \(\);

      intr\_set\_level \(old\_level\);

    \}

#### 3\.2\.1\.3 实现结果

    tests/threads/alarm\_priority通过测试

### <a id="_Toc122817986"></a>3\.2\.2 优先级机制的继续改进

#### 3\.2\.2\.1 实现思路

priority\-change（）：

    void

    test\_priority\_preempt \(void\)

    \{

      /\* This test does not work with the MLFQS\. \*/

      ASSERT \(\!thread\_mlfqs\);

      /\* Make sure our priority is the default\. \*/

      ASSERT \(thread\_get\_priority \(\) == PRI\_DEFAULT\);

      thread\_create \("high\-priority", PRI\_DEFAULT \+ 1, simple\_thread\_func, NULL\);

      msg \("The high\-priority thread should have already completed\."\);

    \}

    static void

    simple\_thread\_func \(void \*aux UNUSED\)

    \{

      int i;

      for \(i = 0; i < 5; i\+\+\)

    \{

      msg \("Thread %s iteration %d", thread\_name \(\), i\);

      thread\_yield \(\);

    \}

      msg \("Thread %s done\!", thread\_name \(\)\);

    \}

    priority\-preempt（）：

    void

    test\_priority\_change \(void\)

    \{

      /\* This test does not work with the MLFQS\. \*/

      ASSERT \(\!thread\_mlfqs\);

      msg \("Creating a high\-priority thread 2\."\);

      thread\_create \("thread 2", PRI\_DEFAULT \+ 1, changing\_thread, NULL\);

      msg \("Thread 2 should have just lowered its priority\."\);

      thread\_set\_priority \(PRI\_DEFAULT \- 2\);

      msg \("Thread 2 should have just exited\."\);

    \}

    static void

    changing\_thread \(void \*aux UNUSED\)

    \{

      msg \("Thread 2 now lowering priority\."\);

      thread\_set\_priority \(PRI\_DEFAULT \- 1\);

      msg \("Thread 2 exiting\."\);

    \}

根据Pintos给到的测试用例来看，我们可以知道，当一个线程的优先级被改变，则需要立即考虑所有线程根据优先级的排序，因此需要在设置优先级函数thread\_set\_priority\(\)中加入thread\_yield \(\)函数以确保每次修改线程优先级后立刻对就绪队列的线程进行重新排序。另外，还需要考虑创建线程时的特殊情况，如果创建的线程优先级高于正在运行的线程的优先级，则需要将正在运行的线程加入就绪队列，并且使新建线程准备运行。

#### 3\.2\.2\.2 实现方法

thread\_set\_priority（）：

```

    /\* Sets the current thread's priority to NEW\_PRIORITY\. \*/

    void

    thread\_set\_priority \(int new\_priority\)

    \{

      thread\_current \(\)\->priority = new\_priority;

      thread\_yield \(\);

    \}

```

然后在thread\_create最后把创建的线程unblock了之后修改成如下代码：

```

    tid\_t

    thread\_create \(const char \*name, int priority,

               thread\_func \*function, void \*aux\) 

    \{

      struct thread \*t;

      struct kernel\_thread\_frame \*kf;

      struct switch\_entry\_frame \*ef;

  struct switch\_threads\_frame \*sf;

  tid\_t tid;

  ASSERT \(function \!= NULL\);

  /\* Allocate thread\. \*/

  t = palloc\_get\_page \(PAL\_ZERO\);

  if \(t == NULL\)

    return TID\_ERROR;

  /\* Initialize thread\. \*/

  init\_thread \(t, name, priority\);

  tid = t\->tid = allocate\_tid \(\);

  t\->ticks\_blocked = 0;

  /\* Stack frame for kernel\_thread\(\)\. \*/

  kf = alloc\_frame \(t, sizeof \*kf\);

  kf\->eip = NULL;

  kf\->function = function;

  kf\->aux = aux;

  /\* Stack frame for switch\_entry\(\)\. \*/

  ef = alloc\_frame \(t, sizeof \*ef\);

  ef\->eip = \(void \(\*\) \(void\)\) kernel\_thread;

  /\* Stack frame for switch\_threads\(\)\. \*/

  sf = alloc\_frame \(t, sizeof \*sf\);

  sf\->eip = switch\_entry;

  sf\->ebp = 0;

  /\* Add to run queue\. \*/

  thread\_unblock \(t\); //不考虑当前运行的进程，新进程在就绪队列中按照优先级排序

  //新加入的语句  

  if \(thread\_current \(\)\->priority < priority\)

  \{

    thread\_yield \(\);//如果当前运行进程优先级还是小于新进程，则要把当前运行进程放入就绪队列

    //由于当前运行进程优先级已经是最高的了，而新创建进程优先级还高的话说明此时新进程优先级最高，则要把它进行执行

  \}

  return tid;

\}

```

#### 3\.2\.2\.3 实现结果

tests/threads/ priority\_change

tests/threads/priority\_fifo

tests/threads/priority\_preempt

通过测试！

### <a id="_Toc122817987"></a>3\.2\.3 通过其他优先级测试程序

#### 3\.2\.3\.1 实现思路

priority\-donate\-one测试用例表明，如果一个线程在获取锁时发现另一个比自己优先级更低的线程已经拥有相同的锁，那么这个线程将会捐赠自己的优先级给另一个线程，即提升另一个线程的优先级与自己相同。

priority\-donate\-multiple与priority\-donate\-multiple2测试用例表明，在恢复线程捐赠后的优先级时，也要考虑其他线程对这个线程的捐赠情况，即需要提供一个数据结果来记录给这个线程捐赠优先级的所有线程。

priority\-donate\-nest测试用例表明，优先级捐赠可以是递归的，因而需要数据结果记录线程正在等待哪个另外线程释放锁。

priority\-donate\-lower测试用例表明，如果线程处于捐赠状态，在修改时线程优先级依然是被捐赠的优先级，但释放锁后线程的优先级变成了修改后的优先级。

priority\-sema和priority\-condvar测试用例表明，需要将信号量的等待队列实现为优先级队列，同时也要将condition的waiters队列实现为优先级队列。

priority\-donate\-chain测试用例表明，释放锁后如果线程没有被捐赠，则需要立即恢复原来的优先级。

总结一下所有测试整合的逻辑：

1\. 在一个线程获取一个锁的时候， 如果拥有这个锁的线程优先级比自己低就提高它的优先级，并且如果这个锁还被别的锁锁着， 将会递归地捐赠优先级， 然后在这个线程释放掉这个锁之后恢复未捐赠逻辑下的优先级。

2\. 如果一个线程被多个线程捐赠， 维持当前优先级为捐赠优先级中的最大值（acquire和release之时）。

3\. 在对一个线程进行优先级设置的时候， 如果这个线程处于被捐赠状态， 则对original\_priority进行设置， 然后如果设置的优先级大于当前优先级， 则改变当前优先级， 否则在捐赠状态取消的时候恢复original\_priority。

4\. 在释放锁对一个锁优先级有改变的时候应考虑其余被捐赠优先级和当前优先级。

5\. 将信号量的等待队列实现为优先级队列。

6\. 将condition的waiters队列实现为优先级队列。

7\. 释放锁的时候若优先级改变则可以发生抢占。

#### 3\.2\.3\.2 实现代码

在thread结构体中加入记录基本优先级、记录线程持有锁和记录线程等待锁的数据结构：

```

struct thread

\{

    int base\_priority;                  /\* Base priority\.新加的 \*/

    struct list locks;                  /\* Locks that the thread is holding\.新加的 \*/

    struct lock \*lock\_waiting;          /\* The lock that the thread is waiting for\. 新加的\*/

\}

```

将上述数据结构在init\_thread中初始化：

```

static void

init\_thread \(struct thread \*t, const char \*name, int priority\)

\{

  t\->base\_priority = priority;

  list\_init \(&t\->locks\);

  t\->lock\_waiting = NULL;

\}

```

在lock结构体中加入记录捐赠和记录最大优先级的数据结构：

```

struct lock 

\{

    struct list\_elem elem;      /\* List element for priority donation\. 新加的\*/

    int max\_priority;          /\* Max priority among the threads acquiring the lock\.新加的 \*/

\};

```

修改synch\.c中的lock\_acquire函数，使其能够以循环的方式实现递归捐赠，并通过修改锁的max\_priority成员，再通过thread\_update\_priority函数更新优先级来实现优先级捐赠：

```

void

lock\_acquire \(struct lock \*lock\)

\{

  struct thread \*current\_thread = thread\_current \(\);

  struct lock \*l;

  enum intr\_level old\_level;

  ASSERT \(lock \!= NULL\);

  ASSERT \(\!intr\_context \(\)\);

  ASSERT \(\!lock\_held\_by\_current\_thread \(lock\)\);

  if \(lock\->holder \!= NULL && \!thread\_mlfqs\)

  \{

    current\_thread\->lock\_waiting = lock;

    l = lock;

    while \(l && current\_thread\->priority > l\->max\_priority\)

    \{

      l\->max\_priority = current\_thread\->priority;

      thread\_donate\_priority \(l\->holder\);

      l = l\->holder\->lock\_waiting;

    \}

  \}

  sema\_down \(&lock\->semaphore\);

  old\_level = intr\_disable \(\);

  current\_thread = thread\_current \(\);

  if \(\!thread\_mlfqs\)

  \{

    current\_thread\->lock\_waiting = NULL;

    lock\->max\_priority = current\_thread\->priority;

    thread\_hold\_the\_lock \(lock\);

  \}

  lock\->holder = current\_thread;

  intr\_set\_level \(old\_level\);

\}

```

实现thread\_donate\_priority和lock\_cmp\_priority，以达到对线程优先级的更新和在队列中位置的重新排布:

```

void thread\_donate\_priority \(struct thread \*t\)

\{

   enum intr\_level old\_level = intr\_disable \(\);

   thread\_update\_priority \(t\);

 

   if \(t\->status == THREAD\_READY\)

   \{

     list\_remove \(&t\->elem\);

     list\_insert\_ordered \(&ready\_list, &t\->elem, thread\_cmp\_priority, NULL\);

   \}

   intr\_set\_level \(old\_level\);

\}

bool lock\_cmp\_priority \(const struct list\_elem \*a, const struct list\_elem \*b, void \*aux UNUSED\)

\{

  return list\_entry \(a, struct lock, elem\)\->max\_priority > list\_entry \(b, struct lock, elem\)\->max\_priority;

\}

```

实现thread\_hold\_the\_lock和lock\_cmp\_priority，以达到对线程拥有锁的记录，同时根据锁记录的线程最大优先级更新当前线程的优先级并重新调度：

```

void thread\_hold\_the\_lock\(struct lock \*lock\)

\{

  enum intr\_level old\_level = intr\_disable \(\);

  list\_insert\_ordered \(&thread\_current \(\)\->locks, &lock\->elem, lock\_cmp\_priority, NULL\);

  if \(lock\->max\_priority > thread\_current \(\)\->priority\)

  \{

    thread\_current \(\)\->priority = lock\->max\_priority;

    thread\_yield \(\);

  \}

  intr\_set\_level \(old\_level\);

\}

bool lock\_cmp\_priority \(const struct list\_elem \*a, const struct list\_elem \*b, void \*aux UNUSED\)

\{

  return list\_entry \(a, struct lock, elem\)\->max\_priority > list\_entry \(b, struct lock, elem\)\->max\_priority;

\}

```

修改lock\_release函数，改变锁的释放行为，并实现thread\_remove\_lock：

```

void lock\_release \(struct lock \*lock\) 

\{

  ASSERT \(lock \!= NULL\);

  ASSERT \(lock\_held\_by\_current\_thread \(lock\)\);

    

  //new code

  if \(\!thread\_mlfqs\)

    thread\_remove\_lock \(lock\);

  lock\->holder = NULL;

  sema\_up \(&lock\->semaphore\);

\}

void thread\_remove\_lock \(struct lock \*lock\)

\{

  enum intr\_level old\_level = intr\_disable \(\);

  list\_remove \(&lock\->elem\);

  thread\_update\_priority \(thread\_current \(\)\);

  intr\_set\_level \(old\_level\);

\}

```

实现thread\_update\_priority，该函数实现释放锁时优先级的变化，如果当前线程还有锁，则获取其拥有锁的max\_priority，如果它大于base\_priority则更新被捐赠的优先级：

```

void thread\_update\_priority \(struct thread \*t\)

\{

  enum intr\_level old\_level = intr\_disable \(\);

  int max\_priority = t\->base\_priority;

  int lock\_priority;

  if \(\!list\_empty \(&t\->locks\)\)

  \{

    list\_sort \(&t\->locks, lock\_cmp\_priority, NULL\);

    lock\_priority = list\_entry \(list\_front \(&t\->locks\), struct lock, elem\)\->max\_priority;

    if \(lock\_priority > max\_priority\)

      max\_priority = lock\_priority;

  \}

  t\->priority = max\_priority;

  intr\_set\_level \(old\_level\);

\}

```

修改thread\_set\_priority函数，完成对新优先级的变换：

```

void thread\_set\_priority \(int new\_priority\)

\{

  if \(thread\_mlfqs\)

    return;

  enum intr\_level old\_level = intr\_disable \(\);

  struct thread \*current\_thread = thread\_current \(\);

  int old\_priority = current\_thread\->priority;

  current\_thread\->base\_priority = new\_priority;

  if \(list\_empty \(&current\_thread\->locks\) || new\_priority > old\_priority\)

  \{

    current\_thread\->priority = new\_priority;

    thread\_yield \(\);

  \}

  intr\_set\_level \(old\_level\);

\}

```

接下来实现sema和condvar这两个优先队列，修改cond\_signal函数，声明并实现比较函数cond\_sema\_cmp\_priority：

```

void cond\_signal \(struct condition \*cond, struct lock \*lock UNUSED\)

\{

  ASSERT \(cond \!= NULL\);

  ASSERT \(lock \!= NULL\);

  ASSERT \(\!intr\_context \(\)\);

  ASSERT \(lock\_held\_by\_current\_thread \(lock\)\);

  if \(\!list\_empty \(&cond\->waiters\)\)

  \{

    list\_sort \(&cond\->waiters, cond\_sema\_cmp\_priority, NULL\);

    sema\_up \(&list\_entry \(list\_pop\_front \(&cond\->waiters\), struct semaphore\_elem, elem\)\->semaphore\);

  \}

\}

bool cond\_sema\_cmp\_priority \(const struct list\_elem \*a, const struct list\_elem \*b, void \*aux UNUSED\)

\{

  struct semaphore\_elem \*sa = list\_entry \(a, struct semaphore\_elem, elem\);

  struct semaphore\_elem \*sb = list\_entry \(b, struct semaphore\_elem, elem\);

  return list\_entry\(list\_front\(&sa\->semaphore\.waiters\), struct thread, elem\)\->priority > list\_entry\(list\_front\(&sb\->semaphore\.waiters\), struct thread, elem\)\->priority;

\}

```

最后把信号量的等待队列实现为优先级队列：

```

void sema\_up \(struct semaphore \*sema\)

\{

  enum intr\_level old\_level;

  ASSERT \(sema \!= NULL\);

  old\_level = intr\_disable \(\);

  if \(\!list\_empty \(&sema\->waiters\)\)

  \{

    list\_sort \(&sema\->waiters, thread\_cmp\_priority, NULL\);

    thread\_unblock \(list\_entry \(list\_pop\_front \(&sema\->waiters\), struct thread, elem\)\);

  \}

  sema\->value\+\+;

  thread\_yield \(\);

  intr\_set\_level \(old\_level\);

\}

void sema\_down \(struct semaphore \*sema\)

\{

  enum intr\_level old\_level;

  ASSERT \(sema \!= NULL\);

  ASSERT \(\!intr\_context \(\)\);

  old\_level = intr\_disable \(\);

  while \(sema\->value == 0\)

    \{

      list\_insert\_ordered \(&sema\->waiters, &thread\_current \(\)\->elem, thread\_cmp\_priority, NULL\);

      thread\_block \(\);

    \}

  sema\->value\-\-;

  intr\_set\_level \(old\_level\);

\}

```

#### 3\.2\.3\.3 实现结果

实现优先级部分pass


## <a id="_Toc122817988"></a>3\.3 实现多级反馈调度

### <a id="_Toc122817989"></a>3\.3\.1 实现思路

在timer\_interrupt中固定一段时间计算更新线程的优先级，这里是每TIMER\_FREQ时间更新一次系统load\_avg和所有线程的recent\_cpu， 每4个timer\_ticks更新一次线程优先级， 每个timer\_tick running线程的recent\_cpu加一， 虽然这里说的是维持64个优先级队列调度， 其本质还是优先级调度， 我们保留之前写的优先级调度代码即可， 去掉优先级捐赠。

### <a id="_Toc122817990"></a>3\.3\.2 实现代码

实现运算逻辑：新建fixed\_point\.h文件，并按照计算公式编写运算程序

```

\#ifndef \_\_THREAD\_FIXED\_POINT\_H

\#define \_\_THREAD\_FIXED\_POINT\_H

/\* Basic definitions of fixed point\. \*/

typedef int fixed\_t;

/\* 16 LSB used for fractional part\. \*/

\#define FP\_SHIFT\_AMOUNT 16

/\* Convert a value to a fixed\-point value\. \*/

\#define FP\_CONST\(A\) \(\(fixed\_t\)\(A << FP\_SHIFT\_AMOUNT\)\)

/\* Add two fixed\-point value\. \*/

\#define FP\_ADD\(A,B\) \(A \+ B\)

/\* Add a fixed\-point value A and an int value B\. \*/

\#define FP\_ADD\_MIX\(A,B\) \(A \+ \(B << FP\_SHIFT\_AMOUNT\)\)

/\* Subtract two fixed\-point value\. \*/

\#define FP\_SUB\(A,B\) \(A \- B\)

/\* Subtract an int value B from a fixed\-point value A\. \*/

\#define FP\_SUB\_MIX\(A,B\) \(A \- \(B << FP\_SHIFT\_AMOUNT\)\)

/\* Multiply a fixed\-point value A by an int value B\. \*/

\#define FP\_MULT\_MIX\(A,B\) \(A \* B\)

/\* Divide a fixed\-point value A by an int value B\. \*/

\#define FP\_DIV\_MIX\(A,B\) \(A / B\)

/\* Multiply two fixed\-point value\. \*/

\#define FP\_MULT\(A,B\) \(\(fixed\_t\)\(\(\(int64\_t\) A\) \* B >> FP\_SHIFT\_AMOUNT\)\)

/\* Divide two fixed\-point value\. \*/

\#define FP\_DIV\(A,B\) \(\(fixed\_t\)\(\(\(\(int64\_t\) A\) << FP\_SHIFT\_AMOUNT\) / B\)\)

/\* Get the integer part of a fixed\-point value\. \*/

\#define FP\_INT\_PART\(A\) \(A >> FP\_SHIFT\_AMOUNT\)

/\* Get the rounded integer of a fixed\-point value\. \*/

\#define FP\_ROUND\(A\) \(A >= 0 ? \(\(A \+ \(1 << \(FP\_SHIFT\_AMOUNT \- 1\)\)\) >> FP\_SHIFT\_AMOUNT\) \\

				: \(\(A \- \(1 << \(FP\_SHIFT\_AMOUNT \- 1\)\)\) >> FP\_SHIFT\_AMOUNT\)\)

\#endif /\* threads/fixed\-point\.h \*/

```

修改timer\_interrupt函数，实现每4个ticks更新一次， 同时保证recent\_cpu自增1的要求：

```

static void

timer\_interrupt \(struct intr\_frame \*args UNUSED\)

\{

  ticks\+\+;

  thread\_tick \(\);

  thread\_foreach \(blocked\_thread\_check, NULL\);

    //new code

  if \(thread\_mlfqs\)

  \{

    mlfqs\_inc\_recent\_cpu\(\);

    if \(ticks % TIMER\_FREQ == 0\)

      mlfqs\_update\_load\_avg\_and\_recent\_cpu\(\);

    else if \(ticks % 4 == 0\)

      mlfqs\_update\_priority\(thread\_current\(\)\);

  \}

\}

```

实现recent\_cpu自增函数：

```

void mlfqs\_inc\_recent\_cpu\(\)

\{

  ASSERT\(thread\_mlfqs\);

  ASSERT\(intr\_context\(\)\);

  struct thread \*cur = thread\_current\(\);

  if \(cur == idle\_thread\)

    return;

  cur\->recent\_cpu = FP\_ADD\_MIX\(cur\->recent\_cpu, 1\);

\}

```

实现mlfqs\_update\_load\_avg\_and\_recent\_cpu函数：

```

void

mlfqs\_update\_load\_avg\_and\_recent\_cpu\(\)

\{

  ASSERT\(thread\_mlfqs\);

  ASSERT\(intr\_context\(\)\);

  size\_t ready\_cnt = list\_size\(&ready\_list\);

  if \(thread\_current\(\) \!= idle\_thread\)

    \+\+ready\_cnt;

  load\_avg = FP\_ADD \(FP\_DIV\_MIX \(FP\_MULT\_MIX \(load\_avg, 59\), 60\), FP\_DIV\_MIX\(FP\_CONST\(ready\_cnt\), 60\)\);

  struct thread \*t;

  struct list\_elem \*e;

  for \(e = list\_begin\(&all\_list\); e \!= list\_end\(&all\_list\); e = list\_next\(e\)\)

  \{

    t = list\_entry\(e, struct thread, allelem\);

    if \(t \!= idle\_thread\)

    \{

      t\->recent\_cpu = FP\_ADD\_MIX \(FP\_MULT \(FP\_DIV \(FP\_MULT\_MIX \(load\_avg, 2\), \\ 

					  FP\_ADD\_MIX \(FP\_MULT\_MIX \(load\_avg, 2\), 1\)\), t\->recent\_cpu\), t\->nice\);

	  mlfqs\_update\_priority\(t\);

	\}

  \}

\}

```


实现mlfqs\_update\_priority函数：

```

void

mlfqs\_update\_priority\(struct thread \*t\)

\{

  ASSERT\(thread\_mlfqs\);

  if \(t == idle\_thread\)

    return;

  t\->priority = FP\_INT\_PART \(FP\_SUB\_MIX \(FP\_SUB \(FP\_CONST \(PRI\_MAX\), \\ 

							 FP\_DIV\_MIX \(t\->recent\_cpu, 4\)\), 2 \* t\->nice\)\);

  if \(t\->priority < PRI\_MIN\)

    t\->priority = PRI\_MIN;

  else if \(t\->priority > PRI\_MAX\)

    t\->priority = PRI\_MAX;

\}

```

在thread结构体中加入成员并在init\_thread中初始化：

```

struct thread

\{

   \.\.\.

   int nice;                           /\* Niceness\. \*/

   fixed\_t recent\_cpu;                 /\* Recent CPU\. \*/ 

   \.\.\.

\}

static void init\_thread \(struct thread \*t, const char \*name, int priority\)

\{

  \.\.\.

  t\->nice = 0;

  t\->recent\_cpu = FP\_CONST \(0\);

  \.\.\.

\}

```

在thread\.c中声明全局变量load\_avg：

    fixed\_t load\_avg;

在thread\_start中初始化load\_avg：

```

void

thread\_start \(void\) 

\{

  load\_avg = FP\_CONST \(0\);

  \.\.\.

\}

```

在thread\.h中包含浮点运算头文件：

```

\#include "fixed\_point\.h"

	在thread\.c中修改thread\_set\_nice、thread\_get\_nice、thread\_get\_load\_avg、thread\_get\_recent\_cpu函数：

/\* Sets the current thread's nice value to NICE\. \*/

void

thread\_set\_nice \(int nice UNUSED\) 

\{

  /\* Solution Code \*/

  thread\_current\(\)\->nice = nice;

  mlfqs\_update\_priority\(thread\_current\(\)\);

  thread\_yield\(\);

\}

/\* Returns the current thread's nice value\. \*/

int

thread\_get\_nice \(void\) 

\{

  /\* Solution Code \*/

  return thread\_current\(\)\->nice;

\}

/\* Returns 100 times the system load average\. \*/

int

thread\_get\_load\_avg \(void\) 

\{

  /\* Solution Code \*/

  return FP\_ROUND \(FP\_MULT\_MIX \(load\_avg, 100\)\);

\}

/\* Returns 100 times the current thread's recent\_cpu value\. \*/

int

thread\_get\_recent\_cpu \(void\) 

\{

  /\* Solution Code \*/

  return FP\_ROUND \(FP\_MULT\_MIX \(thread\_current\(\)\->recent\_cpu, 100\)\);

\}

```

### <a id="_Toc122817991"></a>3\.3\.3 实现结果

27 tests ALL pass

# <a id="_Toc122817992"></a>4\. 实验总结

本次实验主要在ubuntu系统上对Pintos操作系统进行调整测试。我的任务基本都已经实现：

1、实现优先级调度

当一个线程被添加到具有比当前运行的线程更高优先级的就绪列表时，当前线程应该立即将处理器分给新线程中。类似地，当线程正在等待锁，信号量或条件变量时，应首先唤醒优先级最高的等待线程。线程应可以随时提高或降低自己的优先级，但降低其优先级而不再具有最高优先级时必须放弃CPU。

2、实现多级反馈调度

实现多级反馈队列调度程序，减少在系统上运行作业的平均响应时间。这里维持了64个队列， 每个队列对应一个优先级， 从PRI\_MIN到PRI\_MAX。通过一些公式计算来计算出线程当前的优先级， 系统调度的时候会从高优先级队列开始选择线程执行， 这里线程的优先级随着操作系统的运转数据而动态改变。

通过这次的学习，我更好的了解了进程的基本状态反映了进程执行过程的变化。包括就绪状态、执行状态、阻塞状态、终止状态，分别对应了thread状态中的thread\_ready、thread\_running、thread\_block、thread\_dying。贯穿进程调度的整个过程，是进程调度的基础。CPU调度算法\-BSD，较好平衡了现场的不同需求，其中priority的根据recent\_cpu、nice解决。其中recent\_cpu是线程最近使用的CPU时间的估计值。近期recent\_cpu越大优先级越低。

最后，通过完成本次实验，我收获了很多，了解了很多操作系统的知识，并且通过自己亲手做实验与所学的知识进行了更好的融合。这一段时间的学习使我受益良多。