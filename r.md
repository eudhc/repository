
# 操作系统课程设计实验报告


# <a id="_Toc122817971"></a>1\.准备

## <a id="_Toc122817972"></a>1\.1 Ubantu

在VMware中安装Ubuntu 18\.04LTS虚拟机，并进行配置。

##  <a id="_Toc122817973"></a>1\.2 Git

1．下载安装好git之后，使用ssh公钥与gitlab进行连接。

2．使用git clone将三字母仓库克隆到自己的虚拟机上。

3．测试git push命令是否能够正常使用。

## <a id="_Toc122817974"></a>1\.3 Pitnos

1．安装 qemu

    sudo apt\-get install qemu

2．从 Git 库获取最新 Pintos

    git clone git://pintos\-os\.org/pintos\-anon

3．进入pintos/src/utils/pintos\-gdb 用 gedit 打开，编辑 GDBMACROS 变量，将Pintos项目完整路径赋给该变量。

4．gedit 打开 Makefile 文件然后把 LOADLIBES 变量名编辑为 LDLIBS

5．在/src/utils 中输入 make 编译

6．编辑/src/threads/Make\.vars：把 bochs 改为 qemu

7．在/src/threads 运行 make

8．在/utils/pintos：把 bochs 改为 qemu

在/utils/pintos：把 kernel\.bin 改成完整路径

在/utils/pintos：把 qemu 改为 qemu\-system\-x86_64

9．gedit /utils/Pintos.pm：把 loader\.bin 改成完整路径

10\.~/\.bashrc 最后面把上面PATH注释，添加 export PATH=自己路径:$PATH 。

11．打开终端输入 source ~/.bashrc 

12．在 Pintos 输入 pintos run alarm-multiple

# <a id="_Toc122817975"></a>2\.源码分析

## <a id="_Toc122817976"></a>2\.1 背景知识

第一步是阅读和理解初始线程系统的代码。Pintos已经实现了线程创建和线程结束、简单在线程之间切换的调度程序、和同步原语。

当线程创建后，需要创建一个新的上下文用于调度。提供一个将在此上下文中运行的函数，作为thread\_create\(\)的参数。当线程第一次被调度运行时，它从函数的开头开始并且在该上下文中执行。当函数返回时，线程就终止。

在任意一个时刻，只有一个线程在运行，其余线程变为非活动状态。调度程序决定下一步运行哪个线程。当一个线程需要等待另一个线程执行某个操作时，同步原语可以强制上下文切换。

上下文切换的机制位于“threads/switch\.S”中，这是80x86的汇编代码。它可以保存当前正在运行的线程的状态并恢复我们要切换的线程的状态。

## <a id="_Toc122817977"></a>2\.2 源文件

（1）文件夹的功能说明

文件夹     | 功能
-------- | -----
threads  | 基本内核的代码
devices  | IO 设备接口，定时器、键盘、磁盘等代码
lib  | 实现了 C 标准库，该目录代码会与 Pintos kernel 一起编译，用户的程序也要在此目 录下运行。内核程序和用户程序都可以使用 \#include 的方式来引入这个目录下的头 文件

（2）threads/ 文件夹的文件说明
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

（3）devices/ 文件夹的文件说明

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

1.首先进程管理这一项目是从timer_sleep()函数着手，以下是该函数的结构：

    /\* Sleeps for approximately TICKS timer ticks\.  Interrupts must

       be turned on\. \*/

    void timer\_sleep\(int64\_t ticks\)

    \{

      int64\_t start = timer\_ticks\(\); //获取开始的时间

      ASSERT\(intr\_get\_level\(\) == INTR\_ON\);

      while \(timer\_elapsed\(start\) < ticks\) //查看当前时间是否小于设定的睡眠时间

      thread\_yield\(\);                    //将当前线程放入就绪队列，并调度下一个线程

    \}

第5行首先获得了线程休眠的开始时间，timer\_ticks\(\)函数在获取时间的过程中采用了关中断保存程序状态字，然后开中断恢复程序状态字从而防止在执行过程中发生中断。

第7行是当前中断的状态，确保中断是打开的。

2.timer_elapsed()函数，下面是次结构：

    /\* Returns the number of timer ticks elapsed since THEN, which

      should be a value once returned by timer\_ticks\(\)\. \*/

    int64\_t

    timer\_elapsed\(int64\_t then\)

    \{

      return timer\_ticks\(\) \- then;

    \}

可以看到这个函数实际是计算当前线程已经休眠的时间，它把结果返回到timer_sleep()函数，然后while循环判断休眠时间是否达到ticks时间，如果没有就不停的进行thread_yield()。

3.thread_yield()函数：

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

cur指针通过调用thread_current()函数来获取指向页面初始位置的指针，进行了多级嵌套。首先函数获取了esp寄存器的值，esp寄存器是指向栈顶的寄存器，为了获取指向页首的指针，Pintos中一页的大小是2^12，所以1这个数字在二进制下左移12位并取反后，再与esp寄存器中的值相与，就可以获得页首指针。

（2）原子化操作

原子化操作就是关中断和开中断操作，以下两个语句：

    old_level = intr_disable();                //关中断


    intr_set_level(old_level);                 //开中断

基本实现步骤是利用堆栈的push和pop语句可以得到程序状态字寄存器的值，然后利用CLI指令关中断，如果要恢复就把值mov进寄存器然后STI开中断。

（3）线程的切换

如果当前线程不空闲，就把它加入就绪队列，通过指针的改变使其与前后的线程关联起来，形成队列之后就将这一线程修改为就绪状态。

所以可以知道thread_yield()函数是将当前线程放入就绪队列，然后调度下一线程。而timer_sleep()函数便是在限定的时间内，把运行的程序放入就绪队列，然后达到休眠的目的。

由于线程不停的在运行和就绪的状态来回切换，所以会极大的消耗资源。

# <a id="_Toc122817979"></a>3.源码实现

## <a id="_Toc122817980"></a>3.1 timer_sleep()函数的重新实现

### <a id="_Toc122817981"></a>3.1.1 实现思路

由于原本的timer_sleep()函数采用运行和就绪间切换的方法会消耗大量CPU的资源，Pintos提供了线程阻塞这一模式，所以在线程结构体中加入一个用于记录线程睡眠时间的变量。通过使用Pintos的时钟中断，使得每个tick将会执行一次。所以此时每次检测时将记录线程睡眠时间的变量自减1。当变量为0时代表能够唤醒该线程，从而减少资源的开销。

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

（1）改线程结构体，在其中增加记录线程睡眠时间的变量ticks\_blocked

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

（2） 在线程被创建的时候，在thread\_create函数内初始化ticks\_blocked为0：

    t\->ticks\_blocked = 0;

（3） 修改时钟中断处理函数， 在timer\_interrupt中加入线程sleep时间的检测：

    thread\_foreach \(blocked\_thread\_check, NULL\);

（4） 上面的的thread\_foreach就是对每个线程都执行blocked\_thread\_check函数：

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

（5） 然后， 给thread添加一个方法blocked\_thread\_check：

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

到这里timer\_sleep\(\)的唤醒机制就ok了。

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

实现优先级调度的核心思想是： 维持就绪队列为一个优先级队列，就是在插入线程到就绪队列的时候一定要保证这个队列是一个优先级队列。

当下面三种情况我们会把线程放入就绪队列中：

1\. thread\_unblock

2\. init\_thread

3\. thread\_yield

我们只需要在放入线程的过程中维持这个就绪队列是优先级队列。

由于Pintos预置函数list\_insert\_ordered\(\)的存在，可以直接使用这个函数实现线程插入时按照优先级完成，使用我们只需要把在末尾插入线程的函数的语句直接进行替换就可以。

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

优先级比较函数thread\_cmp\_priority\(\):

    /\* priority compare function\. \*/

    bool

    thread\_cmp\_priority \(const struct list\_elem \*a, const struct list\_elem \*b, void \*aux UNUSED\)

    \{

      return list\_entry\(a, struct thread, elem\)\->priority > list\_entry\(b, struct thread, elem\)\->priority;

    \}

调用Pintos预置函数list\_insert\_ordered\(\)替换list\_push\_back\(\)函数，他是在thread\_unblock\(\)中的：

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

根据Pintos给到的测试用例可知，当一个线程的优先级被改变，就要考虑所有线程根据优先级的排序，所以在设置优先级函数thread\_set\_priority\(\)中加入thread\_yield \(\)函数来保证每次修改线程优先级后立刻对就绪队列的线程进行重新排序。还需要考虑特殊情况，当创建的线程优先级高于正在运行的线程的优先级，就要把正在运行的线程加入就绪队列，然后使新建线程准备运行。

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

然后在thread\_create最后把创建的线程unblock了之后把它修改成如下的代码：

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

priority\-donate\-one测试用例表明，如果一个线程在获取锁时发现另一个比自己优先级更低的线程已经拥有相同的锁，那么这个线程将会捐赠自己的优先级给另一个线程，就是把另一个线程的优先级变成跟自己一样。

priority\-donate\-multiple与priority\-donate\-multiple2测试用例表明，在恢复线程捐赠后的优先级时，也要考虑其他线程对这个线程的捐赠情况。

priority\-donate\-nest测试用例表明，优先级捐赠可以是递归的，因而需要数据结果记录线程正在等待哪个另外线程释放锁。

priority\-donate\-lower测试用例表明，如果线程处于捐赠状态，在修改时线程优先级依然是被捐赠的优先级，但释放锁后线程的优先级变成了修改后的优先级。

priority\-sema和priority\-condvar测试用例表明，需要将信号量的等待队列实现为优先级队列，同时也要将condition的waiters队列实现为优先级队列。

priority\-donate\-chain测试用例表明，释放锁后如果线程没有被捐赠，则需要立即恢复原来的优先级。

总结一下测试代码整合的逻辑。

#### 3\.2\.3\.2 实现代码

在thread结构体中加入新的成员变量：记录基本优先级、记录线程持有锁、记录线程等待锁的数据结构：

```

struct thread

\{

    int base\_priority;                  /\* Base priority\.新加的 \*/

    struct list locks;                  /\* Locks that the thread is holding\.新加的 \*/

    struct lock \*lock\_waiting;          /\* The lock that the thread is waiting for\. 新加的\*/

\}

```

把上面新加的在init\_thread中初始化：

```

static void

init\_thread \(struct thread \*t, const char \*name, int priority\)

\{

  t\->base\_priority = priority;

  list\_init \(&t\->locks\);

  t\->lock\_waiting = NULL;

\}

```

在lock结构体中加入新的成员变量：记录捐赠和最大优先级的数据结构：

```

struct lock 

\{

    struct list\_elem elem;      /\* List element for priority donation\. 新加的\*/

    int max\_priority;          /\* Max priority among the threads acquiring the lock\.新加的 \*/

\};

```

修改synch\.c中的lock\_acquire函数，使他能以循环的方式来实现递归捐赠，然后修改锁的max\_priority成员，通过thread\_update\_priority函数来更新优先级从而来实现优先级捐赠：

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

实现thread\_donate\_priority和lock\_cmp\_priority，完成线程优先级的更新以及在队列中位置的重新排布:

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

thread\_hold\_the\_lock和lock\_cmp\_priority，实现对线程拥有锁的记录，同时能够根据锁记录线程的最大优先级来更新当前线程的优先级，从而重新进行调度：

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

修改lock\_release函数，改变锁的释放行为，实现thread\_remove\_lock：

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

实现thread\_update\_priority，完成释放锁时优先级的变化，如果线程还有锁，就获取锁的max\_priority，如果大于base\_priority就要更新被捐赠的优先级：

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

实现sema和condvar优先队列，修改cond\_signal函数，然后声明并实现比较函数cond\_sema\_cmp\_priority：

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

在timer\_interrupt中固定一段时间计算更新线程的优先级，这是每TIMER\_FREQ时间更新一次系统load\_avg和所有线程的recent\_cpu， 每4个timer\_ticks更新一次线程优先级， 每个timer\_tick running线程的recent\_cpu加一， 这里描述是维持64个优先级队列调度， 但是他还是是优先级调度， 去掉优先级捐赠就可以了。

### <a id="_Toc122817990"></a>3\.3\.2 实现代码

实现运算逻辑：新建fixed\_point\.h文件，然后按照计算公式编写运算程序

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

修改timer\_interrupt函数，实现每4个ticks更新一次， 而且要求保证recent\_cpu自增1：

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

在thread结构体中加入新的成员变量然后在init\_thread中初始化：

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


    #include "fixed_point.h"

在thread\.c中修改thread\_set\_nice、thread\_get\_nice、thread\_get\_load\_avg、thread\_get\_recent\_cpu函数：

```

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

本次实验主要在ubuntu系统上对Pintos操作系统进行调整测试。我的任务基本都已经实现。

通过这次的学习，我更好的了解了进程的基本状态反映了进程执行过程的变化。其中包括就绪状态、执行状态、阻塞状态、终止状态，分别对应了thread状态中的thread\_ready、thread\_running、thread\_block、thread\_dying。贯穿进程调度的整个过程，是进程调度的基础。CPU调度算法\-BSD，较好平衡了现场的不同需求。

1.操作系统在我的理解中就是一个介于硬件和软件中的一个东西，他就相当于一个媒介一样，能够帮助我们更好的通过操作软件来实现硬件的操作。然后进程就是指在跑的程序，它有一定的生命周期。线程是比进程更小的一个基本单位，它相当于就是进程的一个实体，可以独立运行。操作系统的基本功能是可以统一的进行pc上资源的管理，对资源进行了一个抽象，是用户和计算机硬件之间的一个润滑剂、工具。操作系统中最近点的就是中断，中断使用可以更好的满足用户的交互需求。进程有五种状态，就绪、阻塞、创建、终止、执行。要实现进程同步，可以使用共享内存、域套接字等的方法。实现线程同步就需要互斥量、读写锁、自旋锁、条件变量的等方法。同时还有死锁饥饿等问题。

2.如果要建立一个线程的模型，因为线程是分为用户线程和内核线程的，主要就是根据他所谓的位置进行分类。他所需要的资源虽然大部分共享进程但是还是需要一些寄存器和栈，线程共享的是地址空间，在同一个地址空间中的线程可以构成一个进程。可以让进程来管理线程也可以是操作系统来管理。

3.首先要解决一个复杂的工程问题，我们先要对这个项目进行一个整体的分析。然后当然是把大工程拆分成一个一个小的工程，简单的问题，然后从0开始一个问题一个解决。就是先利用所学的基本原理对应一个个小的问题，然后逐步进行解决。当然前提是我们能够对这个复杂项目有一个清晰的认识，只有完全分析清楚了才可以进行拆分，从而进行解决。

4.要有试验方案设计的能力，首先必须要对所学的知识内容有一个深入的了解并且掌握，这才具备了能够进行实验的能力，然后要对你要完成的目标有一个清晰的认识，只有这样，你才能够设计出一个合理合规的实验方案，然后进行试验。

5.随着技术时代的发展，我们的社会和环境面临着很大的威胁，我们工科的学生应该挺身而出为着可持续发展做出我们的贡献。首先我们可以利用编程技术对工业等生产尽心自动化编译，使其生产过程更加合理且节约资源、保护环境，从而实现可持续发展。同时利用我们的所学操作系统知识，可以为工业自动化进行更加合理的设计，使其更加符合绿色环保。能够监控污染废料的处理排放，保护资源，保护社会环境，从而为我们的赖以生存的家园进行可持续化发展的保驾护航。

最后，通过完成本次实验，我收获了很多，了解了很多操作系统的知识，并且通过自己亲手做实验与所学的知识进行了更好的融合。这一段时间的学习使我受益良多。