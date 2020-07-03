# Python中的GIL机制详解

大家应该都知道，python有一个GIL（全局解释器锁），用于控制多线程的并发行为。  
`注：GIL不是必须的，可以通过对每个资源单独加锁的方式去掉GIL，也就是将GIL换成更细粒度的锁。`

### GIL锁的实现
Linux上的POSIX线程的实现有如下行为：
* 1、同一个线程多次调用pthread_mutex_lock，在linux中默认类型的锁第二次调用总会堵塞
* 2、一个已经锁住的锁，其他线程调用pthread_mutex_unlock，在linux中默认类型的锁总会被释放

正是由于这些未定义的行为, 并且mutex lock只适用于同步线程对于小段临界区代码的访问，所以GIL的实现没有直接使用原生的`pthread_mutex_lock()/pthread_mutex_unlock()`。

#### GIL的定义
Python的GIL实际是一个<condition, mutex>对, 并用这个条件变量和互斥锁来保护一个locked状态变量
```c++
typedef struct {
    char             locked; /* 0=unlocked, 1=locked */
    /* a <cond, mutex> pair to handle an acquire of a locked lock */
    pthread_cond_t   lock_released;
    pthread_mutex_t  mut;
} pthread_lock;
```
可以看出, locked用来指示是否上锁, 1表示已有线程上锁, 0表示锁空闲。  
而lock_released和mutex来同步对locked的访问。

从GIL的定义来看,线程对GIL的操作本质上就是通过修改locked状态变量来获取或释放GIL。所以主要的操作有两个:

#### 获取GIL锁
```c++
int  PyThread_acquire_lock(PyThread_type_lock lock, int waitflag) 
{
    int success;
    pthread_lock *thelock = (pthread_lock *)lock;
    int status, error = 0;
    status = pthread_mutex_lock( &thelock->mut ); 
    success = thelock->locked == 0;
    if ( !success && waitflag ) { 
        while ( thelock->locked ) {
            status = pthread_cond_wait(&thelock->lock_released,
                                       &thelock->mut);
        }
        success = 1;
    }
    if (success) thelock->locked = 1; 
    status = pthread_mutex_unlock( &thelock->mut ); 
    if (error) success = 0;
    return success; 
}
```
大致流程就是：
* 1、先尝试去获取互斥量mutex，如果获取失败，则循环监控locked状态，等待持有锁的线程释放锁
* 2、如果获取到互斥量，将locked状态置1，表示锁已被该线程持有，其他线程需要等待，然后释放互斥量，让其他线程有机会进入临界区等待上锁

`注：这里的metex作用是获取操作locked变量的权限，而不是获取锁。`

#### 释放GIL锁
```c++
void PyThread_release_lock(PyThread_type_lock lock)
{
    pthread_lock *thelock = (pthread_lock *)lock;
    int status, error = 0;
    status = pthread_mutex_lock( &thelock->mut ); 
    thelock->locked = 0; 
    status = pthread_mutex_unlock( &thelock->mut );
    status = pthread_cond_signal( &thelock->lock_released ); 
}
```
大致流程就是：
* 1、获取互斥量mutex，操作locked变量的权限
* 2、将locked置为0，实际就是释放GIL锁
* 3、释放互斥量mutex，通知其他线程当前线程已经释放GIL锁


### 切换GIL的时机
我们知道，GIL是用来同步多线程使得同一时刻只有一个线程可以解释执行字节码的，显然多线程下，一个线程执行一段时间之后就要释放GIL让其他线程有执行的机会，  
而且从获取与释放GIL的实现来看，只有持有GIL的线程主动释放GIL，其他线程才有机会获取GIL执行自己的任务。那么到底多长时间之后会释放GIL呢？

#####  1、通过判断指令计数器切换GIL

python解释器的主循环代码如下：
```python
{
    ...
    for (;;)
        ...
        if (--_Py_Ticker < 0) { 
            ...
            _Py_Ticker = _Py_CheckInterval;
            ...
            if (interpreter_lock) {
                ...
                PyThread_release_lock(interpreter_lock); 
                PyThread_acquire_lock(interpreter_lock, 1); 
            }
        }
        ...
        switch (opcode) { 
            case: ...
        }
    }
}
```
大致流程就是：
* 1、python的解释器是在一个大的循环中逐个解析字节码指令
* 2、每次循环开始都会检查一下_Py_Ticker的值,这个值可以通过如下查看:
```sh
python -c 'import sys;print(sys.getcheckinterval())'
100
```
默认是100，这个值可以认为是执行的字节码条数的一个计数器，严格上来说应该并不完全等于字节码条数。  
可以看出，这个数字在执行字节码的过程中是递减的，而每次进入一条新的字节码之前都会检查这个数字，当这个数字小于0的时候，就会释放GIL。  
`注：在python3.2的时候已经不是通过指令条数来切换了，而是时间间隔`
```sh
python -c 'import sys;print(sys.getswitchinterval())'
0.005
```

##### 2、IO阻塞之前切换GIL
有这样的场景：  
假如在解析执行字节码的过程中当前线程遇到了一个IO操作而被阻塞，由于只有主动释放GIL，其他线程才有机会运行，由于当前线程已经被阻塞了而无法主动释放锁。  
那么对于这样的场景，python解释器是怎么处理的呢？

Python允许在执行block型的`system call`之前允许其他线程执行`(Py_BEGIN_ALLOW_THREADS)`，然后再重新尝试获取`GIL(Py_END_ALLOW_THREADS)`。  
1、`Py_BEGIN_ALLOW_THREADS`定义:

```c++
#define Py_BEGIN_ALLOW_THREADS { \
                        PyThreadState *_save; \
                        _save = PyEval_SaveThread();

```
`PyEval_SaveThread()`里会释放GIL

2、`Py_END_ALLOW_THREADS`定义：

```c++
#define Py_END_ALLOW_THREADS    PyEval_RestoreThread(_save); \
                }
```
`PyEval_RestoreThread()`里会获取GIL


### GIL之问
##### 1、python有了GIL机制为啥还需要线程同步？
从上面的分析我们知道

GIL 的作用是：对于一个解释器，只能有一个thread在执行bytecode。所以每时每刻只有一条bytecode在被执行一个thread。

GIL保证了bytecode 这层面上是thread safe的。但是如果有个操作比如 x += 1，这个操作需要多个bytecodes操作，在执行这个操作的多条bytecodes期间的时候可能中途就换thread了，这样就出现了data races的情况了。

##### 2、GIL与Java中锁的区别
直接贴几个网上的讨论

[如果GIL是低效的设计，与其对应的什么设计是好的替代方案？](https://www.zhihu.com/question/24702547)
[Why is there no GIL in the Java Virtual Machine? Why does Python need one so bad?](https://stackoverflow.com/questions/991904/why-is-there-no-gil-in-the-java-virtual-machine-why-does-python-need-one-so-bad)
