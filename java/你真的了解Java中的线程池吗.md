# 你真的了解Java中的线程池吗

Java中的线程池，相信大家都接触过或者使用过，它里面到底是怎么运作的，不知道大家有没有去实际了解过？这篇文章将带领大家去看看它内部结构和实现原理。

### 继承关系
```java
public class ThreadPoolExecutor extends AbstractExecutorService {}

public abstract class AbstractExecutorService implements ExecutorService {}

public interface ExecutorService extends Executor {}
```
ThreadPoolExecutor实现的顶层接口是Executor，顶层接口Executor提供了一种思想：将任务提交和任务执行进行解耦。用户无需关注如何创建线程，如何调度线程来执行任务，用户只需提供Runnable对象，将任务的运行逻辑提交到执行器(Executor)中，由Executor框架完成线程的调配和任务的执行部分。
1、ExecutorService接口增加了一些能力：（1）扩充执行任务的能力，补充可以为一个或一批异步任务生成Future的方法；（2）提供了管控线程池的方法，比如停止线程池的运行。
2、AbstractExecutorService则是上层的抽象类，将执行任务的流程串联了起来，保证下层的实现只需关注一个执行任务的方法即可。
3、最下层的实现类ThreadPoolExecutor实现最复杂的运行部分，ThreadPoolExecutor将会一方面维护自身的生命周期，另一方面同时管理线程和任务，使两者良好的结合从而执行并行任务。
ThreadPoolExecutor在内部实际上构建了一个生产者消费者模型，将线程和任务两者解耦，并不直接关联，从而良好的缓冲任务，复用线程。线程池的运行主要分成两部分：任务管理、线程管理。任务管理部分充当生产者的角色，当任务提交后，线程池会判断该任务后续的流转：（1）直接申请线程执行该任务；（2）缓冲到队列中等待线程执行；（3）拒绝该任务。线程管理部分是消费者，它们被统一维护在线程池内，根据任务请求进行线程的分配，当线程执行完任务后则会继续获取新的任务去执行，最终当线程获取不到任务的时候，线程就会被回收。

### 线程池状态
线程池内部使用一个变量ctl来维护两个值：运行状态(runState)和线程数量 (workerCount)。在具体实现中，线程池将运行状态(runState)、线程数量 (workerCount)两个关键参数的维护放在了一起。
ctl是一个AtomicInteger类型，高3位保存runState，低29位保存workerCount，两个变量之间互不干扰。用一个变量去存储两个值，可避免在做相关决策时，出现不一致的情况，不必为了维护两者的一致，而占用锁资源。通过阅读线程池源代码也可以发现，经常出现要同时判断线程池运行状态和线程数量的情况。线程池也提供了若干方法去供用户获得线程池当前的运行状态、线程个数。这里都使用的是位运算的方式，相比于基本运算，速度也会快很多。
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```


### 任务调度机制
一般我们使用线程池，都是直接调用它的execute方法，代码如下：
```java
public void execute(Runnable command) {
	if (command == null)
		throw new NullPointerException();
	/*
	 * Proceed in 3 steps:
	 *
	 * 1. If fewer than corePoolSize threads are running, try to
	 * start a new thread with the given command as its first
	 * task.  The call to addWorker atomically checks runState and
	 * workerCount, and so prevents false alarms that would add
	 * threads when it shouldn't, by returning false.
	 *
	 * 2. If a task can be successfully queued, then we still need
	 * to double-check whether we should have added a thread
	 * (because existing ones died since last checking) or that
	 * the pool shut down since entry into this method. So we
	 * recheck state and if necessary roll back the enqueuing if
	 * stopped, or start a new thread if there are none.
	 *
	 * 3. If we cannot queue task, then we try to add a new
	 * thread.  If it fails, we know we are shut down or saturated
	 * and so reject the task.
	 */
	int c = ctl.get();
	if (workerCountOf(c) < corePoolSize) {
		if (addWorker(command, true))
			return;
		c = ctl.get();
	}
	if (isRunning(c) && workQueue.offer(command)) {
		int recheck = ctl.get();
		if (! isRunning(recheck) && remove(command))
			reject(command);
		else if (workerCountOf(recheck) == 0)
			addWorker(null, false);
	}
	else if (!addWorker(command, false))
		reject(command);
}
```
上面代码的主要逻辑如下：
* 1、首先检测线程池运行状态，如果不是RUNNING，则直接拒绝，线程池要保证在RUNNING的状态下执行任务。
* 2、如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务。
* 3、如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中。
* 4、如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务。
* 5、如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

### 任务缓冲
任务缓冲模块是线程池能够管理任务的核心部分。线程池的本质是对任务和线程的管理，而做到这一点最关键的思想就是将任务和线程两者解耦，不让两者直接关联，才可以做后续的分配工作。线程池中是以生产者消费者模式，通过一个阻塞队列来实现的。阻塞队列缓存任务，工作线程从阻塞队列中获取任务。
```java
private final BlockingQueue<Runnable> workQueue;
```

阻塞队列(BlockingQueue)是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

#### 1. BlockingQueue的核心方法
1、放入数据：
* offer(anObject):表示如果可能的话,将anObject加到BlockingQueue里,即如果BlockingQueue可以容纳,则返回true,否则返回false.（本方法不阻塞当前执行方法的线程）
* offer(E o, long timeout, TimeUnit unit),可以设定等待的时间，如果在指定的时间内，还不能往队列中加入BlockingQueue，则返回失败。
* put(anObject):把anObject加到BlockingQueue里,如果BlockQueue没有空间,则调用此方法的线程被阻断直到BlockingQueue里面有空间再继续.
  2、获取数据：
* poll(time):取走BlockingQueue里排在首位的对象,若不能立即取出,则可以等time参数规定的时间,取不到时返回null;
* poll(long timeout, TimeUnit unit)：从BlockingQueue取出一个队首的对象，如果在指定时间内，队列一旦有数据可取，则立即返回队列中的数据。否则知道时间超时还没有数据可取，返回失败。
* take():取走BlockingQueue里排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到BlockingQueue有新的数据被加入; 
* drainTo():一次性从BlockingQueue获取所有可用的数据对象（还可以指定获取数据的个数）， 通过该方法，可以提升获取数据效率；不需要多次分批加锁或释放锁。

#### 2. 阻塞队列的实现
1. ArrayBlockingQueue：基于数组实现的一个阻塞队列，在创建ArrayBlockingQueue对象时必须制定容量大小。并且可以指定公平性与非公平性，默认情况下为非公平的，即不保证等待时间最长的队列最优先能够访问队列。
2. LinkedBlockingQueue：基于链表实现的一个阻塞队列，在创建LinkedBlockingQueue对象时如果不指定容量大小，则默认大小为Integer.MAX_VALUE。
3. PriorityBlockingQueue：以上2种队列都是先进先出队列，而PriorityBlockingQueue却不是，它会按照元素的优先级对元素进行排序，按照优先级顺序出队，每次出队的元素都是优先级最高的元素。注意，此阻塞队列为无界阻塞队列，即容量没有上限（通过源码就可以知道，它没有容器满的信号标志），前面2种都是有界队列。
4. DelayQueue：基于PriorityQueue，一种延时阻塞队列，DelayQueue中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素。DelayQueue也是一个无界队列，因此往队列中插入数据的操作（生产者）永远不会被阻塞，而只有获取数据的操作（消费者）才会被阻塞。

### 获取待执行任务
由上文的任务分配部分可知，任务的执行有两种可能：一种是任务直接由新创建的线程执行。另一种是线程从任务队列中获取任务然后执行，执行完任务的空闲线程会再次去从队列中申请任务再去执行。第一种情况仅出现在线程初始创建的时候，第二种是线程获取任务绝大多数的情况。
线程需要从任务缓存模块中不断地取任务执行，帮助线程从阻塞队列中获取任务，实现线程管理模块和任务管理模块之间的通信。这部分策略由getTask方法实现:
```java
private Runnable getTask() {
	boolean timedOut = false; // Did the last poll() time out?

	for (;;) {
		int c = ctl.get();
		int rs = runStateOf(c);

		// Check if queue empty only if necessary.
		if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
			decrementWorkerCount();
			return null;
		}

		int wc = workerCountOf(c);

		// Are workers subject to culling?
		boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

		if ((wc > maximumPoolSize || (timed && timedOut))
			&& (wc > 1 || workQueue.isEmpty())) {
			if (compareAndDecrementWorkerCount(c))
				return null;
			continue;
		}

		try {
			Runnable r = timed ?
				workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
				workQueue.take();
			if (r != null)
				return r;
			timedOut = true;
		} catch (InterruptedException retry) {
			timedOut = false;
		}
	}
}
```
getTask这部分进行了多次判断，为的是控制线程的数量，使其符合线程池的状态。如果线程池现在不应该持有那么多线程，则会返回null值。工作线程Worker会不断接收新任务去执行，而当工作线程Worker接收不到任务的时候，就会开始被回收。

### 任务拒绝
任务拒绝模块是线程池的保护部分，线程池有一个最大的容量，当线程池的任务缓存队列已满，并且线程池中的线程数目达到maximumPoolSize时，就需要拒绝掉该任务，采取任务拒绝策略，保护线程池。
拒绝策略是一个接口，其设计如下：
```java
public interface RejectedExecutionHandler {

    /**
     * Method that may be invoked by a {@link ThreadPoolExecutor} when
     * {@link ThreadPoolExecutor#execute execute} cannot accept a
     * task.  This may occur when no more threads or queue slots are
     * available because their bounds would be exceeded, or upon
     * shutdown of the Executor.
     *
     * <p>In the absence of other alternatives, the method may throw
     * an unchecked {@link RejectedExecutionException}, which will be
     * propagated to the caller of {@code execute}.
     *
     * @param r the runnable task requested to be executed
     * @param executor the executor attempting to execute this task
     * @throws RejectedExecutionException if there is no remedy
     */
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

线程池提供了以下几种策略：
* 1、AbortPolicy：该策略是线程池的默认策略。使用该策略时，如果线程池队列满了丢掉这个任务并且抛出RejectedExecutionException异常。 
* 2、DiscardPolicy：如果线程池队列满了，会直接丢掉这个任务并且不会有任何异常。
* 3、DiscardOldestPolicy：这个策略从字面上也很好理解，丢弃最老的。也就是说如果队列满了，会将最早进入队列的任务删掉腾出空间，再尝试加入队列。 因为队列是队尾进，队头出，所以队头元素是最老的，因此每次都是移除对头元素后再尝试入队。
* 4、CallerRunsPolicy：使用此策略，如果添加到线程池失败，那么调用线程（提交任务的线程）会自己去执行该任务，不会等待线程池中的线程去执行。

如果以上策略都不符合业务场景，那么可以自己定义一个拒绝策略，只要实现`RejectedExecutionHandler`接口，并且实现`rejectedExecution`方法就可以了。具体的逻辑就在`rejectedExecution`方法里去定义就OK了。

### 工作线程
线程池提供了一个工作线程任务的概念，对应实现类如下：
```java
private final class Worker
	extends AbstractQueuedSynchronizer
	implements Runnable
{
     Worker(Runnable firstTask) {
		setState(-1); // inhibit interrupts until runWorker
		this.firstTask = firstTask;
		this.thread = getThreadFactory().newThread(this);
	}
}
```
Worker这个工作线程，实现了Runnable接口，并持有一个线程thread，一个初始化的任务firstTask。
thread是在调用构造方法时通过ThreadFactory来创建的线程，可以用来执行任务；
firstTask用它来保存传入的第一个任务，这个任务可以有也可以为null。如果这个值是非空的，那么线程就会在启动初期立即执行这个任务，也就对应核心线程创建时的情况；如果这个值是null，那么就需要创建一个线程去执行任务列表（workQueue）中的任务，也就是非核心线程的创建。

线程池需要管理线程的生命周期，需要在线程长时间不运行的时候进行回收。线程池使用一张Hash表去持有线程的引用，这样可以通过添加引用、移除引用这样的操作来控制线程的生命周期。这个时候重要的就是如何判断线程是否在运行。
```java
private final HashSet<Worker> workers = new HashSet<Worker>();
```

Worker是通过继承AQS，使用AQS来实现独占锁这个功能。没有使用可重入锁ReentrantLock，而是使用AQS，为的就是实现不可重入的特性去反应线程现在的执行状态。

1.lock方法一旦获取了独占锁，表示当前线程正在执行任务中。 2.如果正在执行任务，则不应该中断线程。 3.如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可以对该线程进行中断。 4.线程池在执行shutdown方法或tryTerminate方法时会调用interruptIdleWorkers方法来中断空闲的线程，interruptIdleWorkers方法会使用tryLock方法来判断线程池中的线程是否是空闲状态；如果线程是空闲状态则可以安全回收。

### 执行任务
执行任务也就是执行工作线程，也就是Runnable的run方法，具体就是ThreadPoolExecutor的runWorker()方法：
```java
final void runWorker(Worker w) {
	Thread wt = Thread.currentThread();
	Runnable task = w.firstTask;
	w.firstTask = null;
	w.unlock(); // allow interrupts
	boolean completedAbruptly = true;
	try {
		while (task != null || (task = getTask()) != null) {
			w.lock();
			// If pool is stopping, ensure thread is interrupted;
			// if not, ensure thread is not interrupted.  This
			// requires a recheck in second case to deal with
			// shutdownNow race while clearing interrupt
			if ((runStateAtLeast(ctl.get(), STOP) ||
				 (Thread.interrupted() &&
				  runStateAtLeast(ctl.get(), STOP))) &&
				!wt.isInterrupted())
				wt.interrupt();
			try {
				beforeExecute(wt, task);
				Throwable thrown = null;
				try {
					task.run();
				} catch (RuntimeException x) {
					thrown = x; throw x;
				} catch (Error x) {
					thrown = x; throw x;
				} catch (Throwable x) {
					thrown = x; throw new Error(x);
				} finally {
					afterExecute(task, thrown);
				}
			} finally {
				task = null;
				w.completedTasks++;
				w.unlock();
			}
		}
		completedAbruptly = false;
	} finally {
		processWorkerExit(w, completedAbruptly);
	}
}
```
上面代码的主要逻辑如下：
* 1.while循环不断地通过getTask()方法获取任务。
* 2.getTask()方法从阻塞队列中取任务。 
* 3.如果线程池正在停止，那么要保证当前线程是中断状态，否则要保证当前线程不是中断状态。
* 4.执行任务。 
* 5.如果getTask结果为null则跳出循环，执行processWorkerExit()方法，销毁线程。

### 线程回收
线程池中线程的销毁依赖JVM自动的回收，线程池做的工作是根据当前线程池的状态维护一定数量的线程引用，防止这部分线程被JVM回收，当线程池决定哪些线程需要回收时，只需要将其引用消除即可。Worker被创建出来后，就会不断地进行轮询，然后获取任务去执行，核心线程可以无限等待获取任务，非核心线程要限时获取任务。当Worker无法获取到任务，也就是获取的任务为空时，循环会结束，Worker会主动消除自身在线程池内的引用。
```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
	if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
		decrementWorkerCount();

	final ReentrantLock mainLock = this.mainLock;
	mainLock.lock();
	try {
		completedTaskCount += w.completedTasks;
		workers.remove(w);
	} finally {
		mainLock.unlock();
	}

	tryTerminate();

	int c = ctl.get();
	if (runStateLessThan(c, STOP)) {
		if (!completedAbruptly) {
			int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
			if (min == 0 && ! workQueue.isEmpty())
				min = 1;
			if (workerCountOf(c) >= min)
				return; // replacement not needed
		}
		addWorker(null, false);
	}
}
```
事实上，在这个方法中，将线程引用移出线程池就已经结束了线程销毁的部分。但由于引起线程销毁的可能性有很多，线程池还要判断是什么引发了这次销毁，是否要改变线程池的现阶段状态，是否要根据新状态，重新分配线程。