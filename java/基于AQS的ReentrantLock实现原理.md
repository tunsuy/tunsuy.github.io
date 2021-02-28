# 基于AQS的ReentrantLock实现原理

Java中的大部分同步类（Lock、Semaphore、ReentrantLock等）都是基于AbstractQueuedSynchronizer（简称为AQS）实现的。AQS是一种提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列模型的简单框架。

### ReentrantLock
ReentrantLock是可重入锁，指的是一个线程能够对一个临界资源重复加锁。
ReentrantLock支持公平锁和非公平锁，并且ReentrantLock的底层就是由AQS来实现的。那么ReentrantLock是如何通过公平锁和非公平锁与AQS关联起来呢？ 
ReentrantLock提供了一个Sync内部类，该类继承了AQS，ReentrantLock的公平锁和非公平锁就是通过继承Sync来跟AQS关联起来的。

我们着重从这两者的加锁过程来理解一下它们与AQS之间的关系。

### 可重入
AQS提供了一个同步状态State来控制整体可重入的情况。State是Volatile修饰的，用于保证一定的可见性和有序性。
```java
AbstractQueuedSynchronizer

private volatile int state;
```
因此ReentrantLock也就具有了可重入的特性。

State这个字段主要的过程：
* 1、State初始化的时候为0，表示没有任何线程持有锁。
* 2、当有线程持有该锁时，值就会在原来的基础上+1，同一个线程多次获得锁是，就会多次+1，这里就是可重入的概念。
* 3、解锁也是对这个字段-1，一直到0，此线程对锁释放。

### 公平锁 VS 非公平锁
```java
static final class NonfairSync extends Sync {
	private static final long serialVersionUID = 7316153563782823691L;

	/**
	 * Performs lock.  Try immediate barge, backing up to normal
	 * acquire on failure.
	 */
	final void lock() {
		if (compareAndSetState(0, 1))
			setExclusiveOwnerThread(Thread.currentThread());
		else
			acquire(1);
	}

	protected final boolean tryAcquire(int acquires) {
		return nonfairTryAcquire(acquires);
	}
}
```
上面代码的逻辑为：
* 1、若通过CAS设置变量State（同步状态）成功，也就是获取锁成功，则将当前线程设置为独占线程。
* 2、若通过CAS设置变量State（同步状态）失败，也就是获取锁失败，则进入Acquire方法进行后续处理。

第一步很好理解，但第二步获取锁失败后，后续的处理策略是怎么样的呢？有以下两种可能：
* (1) 将当前线程获锁结果设置为失败，获取锁流程结束。这种设计会极大降低系统的并发度，并不满足我们实际的需求。所以就需要下面这种流程，也就是AQS框架的处理流程。
* (2) 存在某种排队等候机制，线程继续等待，仍然保留获取锁的可能，获取锁流程仍在继续。

对于第二种情况，就有如下几种疑问：
* 既然说到了排队等候机制，那么就一定会有某种队列形成，这样的队列是什么数据结构呢？
* 处于排队等候机制中的线程，什么时候可以有机会获取锁呢？
* 如果处于排队等候机制中的线程一直无法获取锁，还是需要一直等待吗，还是有别的策略来解决这一问题？
  这些问题我们先记下来，后面慢慢讲解。

同样的，对于公平锁
```java
static final class FairSync extends Sync {
	private static final long serialVersionUID = -3000897897090466540L;

	final void lock() {
		acquire(1);
	}

	/**
	 * Fair version of tryAcquire.  Don't grant access unless
	 * recursive call or no waiters or is first.
	 */
	protected final boolean tryAcquire(int acquires) {
		final Thread current = Thread.currentThread();
		int c = getState();
		if (c == 0) {
			if (!hasQueuedPredecessors() &&
				compareAndSetState(0, acquires)) {
				setExclusiveOwnerThread(current);
				return true;
			}
		}
		else if (current == getExclusiveOwnerThread()) {
			int nextc = c + acquires;
			if (nextc < 0)
				throw new Error("Maximum lock count exceeded");
			setState(nextc);
			return true;
		}
		return false;
	}
}
```

ReentrantLock默认使用非公平锁，也可以通过构造器来显示的指定使用公平锁。
通过上面的源代码对比，我们可以明显的看出公平锁与非公平锁的lock()方法唯一的区别就在于公平锁在获取同步状态时多了一个限制条件：hasQueuedPredecessors()。
该方法主要做一件事情：主要是判断当前线程是否位于同步队列中的第一个。如果是则返回true，否则返回false。

综上，公平锁就是通过同步队列来实现多个线程按照申请锁的顺序来获取锁，从而实现公平的特性。非公平锁加锁时不考虑排队等待问题，直接尝试获取锁，所以存在后申请却先获得锁的情况。


到了这里，我看到了，公平锁和非公平锁的加锁流程，虽然流程上有一定的不同，但是都调用了Acquire方法，而Acquire方法是FairSync和UnfairSync的父类AQS中的核心方法。
对于上边提到的问题，都需要由acquire来解答，而Acquire方法是在类AbstractQueuedSynchronizer中实现的。下面我们会对AQS以及ReentrantLock和AQS的关联做详细介绍。

### AQS架构
总的来说，AQS框架共分为五层，自上而下由浅入深，从AQS对外暴露的API到底层基础数据。
当有自定义同步器接入时，只需重写第一层所需要的部分方法即可，不需要关注底层具体的实现流程。当自定义同步器进行加锁或者解锁操作时，先经过第一层的API进入AQS内部方法，然后经过第二层进行锁的获取，接着对于获取锁失败的流程，进入第三层和第四层的等待队列处理，而这些处理方式均依赖于第五层的基础数据提供层。

AQS核心思想是，如果被请求的共享资源空闲，那么就将当前请求资源的线程设置为有效的工作线程，将共享资源设置为锁定状态；如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配。这个机制主要用的是CLH队列的变体实现的，将暂时获取不到锁的线程加入到队列中。

CLH：Craig、Landin and Hagersten队列，是单向链表，AQS中的队列是CLH变体的虚拟双向队列（FIFO），AQS是通过将每条请求共享资源的线程封装成一个节点来实现锁的分配。
AQS使用一个Volatile的int类型的成员变量来表示同步状态，通过内置的FIFO队列来完成资源获取的排队工作，通过CAS完成对State值的修改。

#### 1、数据结构——Node
AQS中定义了一个节点类的Node，下面是主要的几个方法和属性值的含义：
* waitStatus：当前节点在队列中的状态
* thread：表示处于该节点的线程
* prev：前驱指针
* predecessor：返回前驱节点，没有的话抛出npe
* nextWaiter：指向下一个处于CONDITION状态的节点（由于本篇文章不讲述Condition Queue队列，这个指针不多介绍）
* next：后继指针

#### 2、锁的模式
* SHARED：表示线程以共享的模式等待锁
* EXCLUSIVE：表示线程正在以独占的方式等待锁

#### 3、线程节点状态
当前节点在队列中的状态waitStatus有如下几种情况：
* 0 当一个Node被初始化的时候的默认值
* CANCELLED：为1，表示线程获取锁的请求已经取消了
* CONDITION：为-2，表示节点在等待队列中，节点线程等待唤醒
* PROPAGATE：为-3，当前线程处在SHARED情况下，该字段才会使用
* SIGNAL：为-1，表示线程已经准备好了，就等资源释放了

### 自定义同步器
AQS提供了大量用于自定义同步器实现的Protected方法。自定义同步器实现的相关方法也只是为了通过修改State字段来实现多线程的独占模式或者共享模式。自定义同步器需要实现以下方法（ReentrantLock需要实现的方法如下，并不是全部）：
* protected boolean isHeldExclusively()：该线程是否正在独占资源。只有用到Condition才需要去实现它。
* protected boolean tryAcquire(int arg)：独占方式。arg为获取锁的次数，尝试获取资源，成功则返回True，失败则返回False。
* protected boolean tryRelease(int arg)：独占方式。arg为释放锁的次数，尝试释放资源，成功则返回True，失败则返回False。
* protected int tryAcquireShared(int arg)：共享方式。arg为获取锁的次数，尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
* protected boolean tryReleaseShared(int arg)：共享方式。arg为释放锁的次数，尝试释放资源，如果释放后允许唤醒后续等待结点返回True，否则返回False。

一般来说，自定义同步器要么是独占方式，要么是共享方式，它们也只需实现`tryAcquire-tryRelease`、`tryAcquireShared-tryReleaseShared`中的一种即可。AQS也支持自定义同步器同时实现独占和共享两种方式，如`ReentrantReadWriteLock`。`ReentrantLock`是独占锁，所以实现了`tryAcquire-tryRelease`。

#### 1、获取锁
下面我们继续回到文章开头的ReentrantLock非公平锁的lock方法：
```java
ReentrantLock.java
final void lock() {
	if (compareAndSetState(0, 1))
		setExclusiveOwnerThread(Thread.currentThread());
	else
		acquire(1);
}

AbstractQueuedSynchronizer.java
public final void acquire(int arg) {
	if (!tryAcquire(arg) &&
		acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		selfInterrupt();
}

ReentrantLock.java
protected final boolean tryAcquire(int acquires) {
	return nonfairTryAcquire(acquires);
}
final boolean nonfairTryAcquire(int acquires) {
	final Thread current = Thread.currentThread();
	int c = getState();
	if (c == 0) {
		if (compareAndSetState(0, acquires)) {
			setExclusiveOwnerThread(current);
			return true;
		}
	}
	else if (current == getExclusiveOwnerThread()) {
		int nextc = c + acquires;
		if (nextc < 0) // overflow
			throw new Error("Maximum lock count exceeded");
		setState(nextc);
		return true;
	}
	return false;
}
```
上面代码的逻辑为：
可以看出，这里只是AQS的简单实现，具体获取锁的实现方法是由各自的公平锁和非公平锁单独实现的（以ReentrantLock为例）。如果该方法返回了True，则说明当前线程获取锁成功，就不用往后执行了；如果获取失败，就需要加入到等待队列中。下面会详细解释线程是何时以及怎样被加入进等待队列中的。
addWaiter方法其实就是把对应的线程以Node的数据结构形式加入到双端队列里，返回的是一个包含该线程的Node。而这个Node会作为参数，进入到acquireQueued方法中。acquireQueued方法可以对排队中的线程进行“获锁”操作。

总的来说，一个线程获取锁失败了，被放入等待队列，acquireQueued会把放入队列中的线程不断去获取锁，直到获取成功或者不再需要获取（中断）。

#### 2、加入队列
获取锁失败后，会执行addWaiter(Node.EXCLUSIVE)加入等待队列，具体实现方法如下：
```java
private Node addWaiter(Node mode) {
	Node node = new Node(Thread.currentThread(), mode);
	// Try the fast path of enq; backup to full enq on failure
	Node pred = tail;
	if (pred != null) {
		node.prev = pred;
		if (compareAndSetTail(pred, node)) {
			pred.next = node;
			return node;
		}
	}
	enq(node);
	return node;
}

```
上面代码的逻辑为：
* 通过当前的线程和锁模式新建一个节点。
* Pred指针指向尾节点Tail。
* 将New中Node的Prev指针指向Pred。
* 通过compareAndSetTail方法，完成尾节点的设置。这个方法主要是对tailOffset和Expect进行比较，如果tailOffset的Node和Expect的Node地址是相同的，那么设置Tail的值为Update的值。

AQS中定义了一些表示偏移量的静态变量
```java
static {
	try {
		stateOffset = unsafe.objectFieldOffset
			(AbstractQueuedSynchronizer.class.getDeclaredField("state"));
		headOffset = unsafe.objectFieldOffset
			(AbstractQueuedSynchronizer.class.getDeclaredField("head"));
		tailOffset = unsafe.objectFieldOffset
			(AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
		waitStatusOffset = unsafe.objectFieldOffset
			(Node.class.getDeclaredField("waitStatus"));
		nextOffset = unsafe.objectFieldOffset
			(Node.class.getDeclaredField("next"));

	} catch (Exception ex) { throw new Error(ex); }
}
```
从AQS的静态代码块可以看出，都是获取一个对象的属性相对于该对象在内存当中的偏移量，这样我们就可以根据这个偏移量在对象内存当中找到这个属性。tailOffset指的是tail对应的偏移量，所以这个时候会将new出来的Node置为当前队列的尾节点。同时，由于是双向链表，也需要将前一个节点指向尾节点。

如果Pred指针是Null（说明等待队列中没有元素），或者当前Pred指针和Tail指向的位置不同（说明被别的线程已经修改），就需要看一下Enq的方法。
```java
private Node enq(final Node node) {
	for (;;) {
		Node t = tail;
		if (t == null) { // Must initialize
			if (compareAndSetHead(new Node()))
				tail = head;
		} else {
			node.prev = t;
			if (compareAndSetTail(t, node)) {
				t.next = node;
				return t;
			}
		}
	}
}

```
如果没有被初始化，需要进行初始化一个头结点出来。但请注意，初始化的头结点并不是当前线程节点，而是调用了无参构造函数的节点。如果经历了初始化或者并发导致队列中有元素，则与之前的方法相同。其实，addWaiter就是一个在双端链表添加尾节点的操作，需要注意的是，双端链表的头结点是一个无参构造函数的头结点。

对于公平锁来说，秉承先到先得的原则，因此在请求锁的时候，需要有如下判断：
```java
public final boolean hasQueuedPredecessors() {
	// The correctness of this depends on head being initialized
	// before tail and on head.next being accurate if the current
	// thread is first in queue.
	Node t = tail; // Read fields in reverse initialization order
	Node h = head;
	Node s;
	return h != t &&
		((s = h.next) == null || s.thread != Thread.currentThread());
}
```
hasQueuedPredecessors是公平锁加锁时判断等待队列中是否存在有效节点的方法。如果返回False，说明当前线程可以争取共享资源；如果返回True，说明队列中存在有效节点，当前线程必须加入到等待队列中。
看到这里，我们理解一下h != t && ((s = h.next) == null || s.thread != Thread.currentThread());为什么要判断的头结点的下一个节点？第一个节点储存的数据是什么？
###### 双向链表中，第一个节点为虚节点，其实并不存储任何信息，只是占位。真正的第一个有数据的节点，是在第二个节点开始的。当h != t时： 如果(s = h.next) == null，等待队列正在有线程进行初始化，但只是进行到了Tail指向Head，没有将Head指向Tail，此时队列中有元素，需要返回True（这块具体见下边代码分析）。 如果(s = h.next) != null，说明此时队列中至少有一个有效节点。如果此时s.thread == Thread.currentThread()，说明等待队列的第一个有效节点中的线程与当前线程相同，那么当前线程是可以获取资源的；如果s.thread != Thread.currentThread()，说明等待队列的第一个有效节点线程与当前线程不同，当前线程必须加入进等待队列。

#### 3、自旋等待
继续回到上面获取锁的代码:
```java
AbstractQueuedSynchronizer

public final void acquire(int arg) {
	if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		selfInterrupt();
}
```
上一节讲解了addWaiter，这里开始分析acquireQueued源码：
```java
AbstractQueuedSynchronizer

final boolean acquireQueued(final Node node, int arg) {
	// 标记是否成功拿到资源
	boolean failed = true;
	try {
		// 标记等待过程中是否中断过
		boolean interrupted = false;
		// 开始自旋，要么获取锁，要么中断
		for (;;) {
			// 获取当前节点的前驱节点
			final Node p = node.predecessor();
			// 如果p是头结点，说明当前节点在真实数据队列的首部，就尝试获取锁（别忘了头结点是虚节点）
			if (p == head && tryAcquire(arg)) {
				// 获取锁成功，头指针移动到当前node
				setHead(node);
				p.next = null; // help GC
				failed = false;
				return interrupted;
			}
			// 说明p为头节点且当前没有获取到锁（可能是非公平锁被抢占了）或者是p不为头结点，这个时候就要判断当前node是否要被阻塞（被阻塞条件：前驱节点的waitStatus为-1），防止无限循环浪费资源。具体两个方法下面细细分析
			if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
				interrupted = true;
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}

// 靠前驱节点判断当前线程是否应该被阻塞
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
	// 获取头结点的节点状态
	int ws = pred.waitStatus;
	// 说明头结点处于唤醒状态
	if (ws == Node.SIGNAL)
		return true; 
	// 通过枚举值我们知道waitStatus>0是取消状态
	if (ws > 0) {
		do {
			// 循环向前查找取消节点，把取消节点从队列中剔除
			node.prev = pred = pred.prev;
		} while (pred.waitStatus > 0);
		pred.next = node;
	} else {
		// 设置前任节点等待状态为SIGNAL
		compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
	}
	return false;
}
```
总的来说，跳出当前循环的条件是当“前置节点是头结点，且当前线程获取锁成功”。为了防止因死循环导致CPU资源被浪费，我们会判断前置节点的状态来决定是否要将当前线程挂起


#### 4、取消自旋等待
同样的，在acquireQueued方法中的Finally代码中的`cancelAcquire`：
```java
AbstractQueuedSynchronizer

private void cancelAcquire(Node node) {
  // 将无效节点过滤
	if (node == null)
		return;
  // 设置该节点不关联任何线程，也就是虚节点
	node.thread = null;
	Node pred = node.prev;
  // 通过前驱节点，跳过取消状态的node
	while (pred.waitStatus > 0)
		node.prev = pred = pred.prev;
  // 获取过滤后的前驱节点的后继节点
	Node predNext = pred.next;
  // 把当前node的状态设置为CANCELLED
	node.waitStatus = Node.CANCELLED;
  // 如果当前节点是尾节点，将从后往前的第一个非取消状态的节点设置为尾节点
  // 更新失败的话，则进入else，如果更新成功，将tail的后继节点设置为null
	if (node == tail && compareAndSetTail(node, pred)) {
		compareAndSetNext(pred, predNext, null);
	} else {
		int ws;
    // 如果当前节点不是head的后继节点，1:判断当前节点前驱节点的是否为SIGNAL，2:如果不是，则把前驱节点设置为SINGAL看是否成功
    // 如果1和2中有一个为true，再判断当前节点的线程是否为null
    // 如果上述条件都满足，把当前节点的前驱节点的后继指针指向当前节点的后继节点
		if (pred != head && ((ws = pred.waitStatus) == Node.SIGNAL || (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) && pred.thread != null) {
			Node next = node.next;
			if (next != null && next.waitStatus <= 0)
				compareAndSetNext(pred, predNext, next);
		} else {
      // 如果当前节点是head的后继节点，或者上述条件不满足，那就唤醒当前节点的后继节点
			unparkSuccessor(node);
		}
		node.next = node; // help GC
	}
}
```
上面代码的逻辑为：
获取当前节点的前驱节点，如果前驱节点的状态是CANCELLED，那就一直往前遍历，找到第一个waitStatus <= 0的节点，将找到的Pred节点和当前Node关联，将当前Node设置为CANCELLED。
根据当前节点的位置，考虑以下三种情况：
* (1) 当前节点是尾节点。
* (2) 当前节点是Head的后继节点。
* (3) 当前节点不是Head的后继节点，也不是尾节点。


### 解锁
ReentrantLock在解锁的时候，并不区分公平锁和非公平锁，因此unlock方法在ReentrantLock类中：
```java
ReentrantLock

public void unlock() {
	sync.release(1);
}
```
可以看到，本质释放锁的地方，是通过框架来完成的。
```java
AbstractQueuedSynchronizer

public final boolean release(int arg) {
	// 自定义的tryRelease如果返回true，说明该锁没有被任何线程持有
	if (tryRelease(arg)) {
		Node h = head;
		if (h != null && h.waitStatus != 0)
			unparkSuccessor(h);
		return true;
	}
	return false;
}
```
这里的tryRelease调用的子类实现的方法，也就是ReentrantLock类，因为是释放锁是不区分公平和不公平的，因此这里的tryRelease的实现在公平锁和非公平锁的共同父类Sync中:
```java
ReentrantLock.Sync

// 方法返回当前锁是不是没有被线程持有
protected final boolean tryRelease(int releases) {
	// 减少可重入次数
	int c = getState() - releases;
	// 当前线程不是持有锁的线程，抛出异常
	if (Thread.currentThread() != getExclusiveOwnerThread())
		throw new IllegalMonitorStateException();
	boolean free = false;
	// 如果持有线程全部释放，将当前独占锁所有线程设置为null，并更新state
	if (c == 0) {
		free = true;
		setExclusiveOwnerThread(null);
	}
	setState(c);
	return free;
}
```

### 唤醒其他线程
```java
AbstractQueuedSynchronizer

private void unparkSuccessor(Node node) {
	// 获取头结点waitStatus
	int ws = node.waitStatus;
	if (ws < 0)
		compareAndSetWaitStatus(node, ws, 0);
	// 获取当前节点的下一个节点
	Node s = node.next;
	// 如果下个节点是null或者下个节点被cancelled，就找到队列最开始的非cancelled的节点
	if (s == null || s.waitStatus > 0) {
		s = null;
		// 就从尾部节点开始找，到队首，找到队列第一个waitStatus<0的节点。
		for (Node t = tail; t != null && t != node; t = t.prev)
			if (t.waitStatus <= 0)
				s = t;
	}
	// 如果当前节点的下个节点不为空，而且状态<=0，就把当前节点unpark
	if (s != null)
		LockSupport.unpark(s.thread);
}
```

唤醒后，会执行return Thread.interrupted();，这个函数返回的是当前执行线程的中断状态，并清除。
```java
AbstractQueuedSynchronizer

private final boolean parkAndCheckInterrupt() {
	LockSupport.park(this);
	return Thread.interrupted();
}
```
再回到acquireQueued代码，当parkAndCheckInterrupt返回True或者False的时候，interrupted的值不同，但都会执行下次循环。如果这个时候获取锁成功，就会把当前interrupted返回。

我们再回到acquire方法:
```java
AbstractQueuedSynchronizer

public final void acquire(int arg) {
	if (!tryAcquire(arg) &&
		acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		selfInterrupt();
}
```
如果acquireQueued为True，就会执行selfInterrupt方法。
```java
AbstractQueuedSynchronizer

static void selfInterrupt() {
	Thread.currentThread().interrupt();
}
```
该方法其实是为了中断线程。但为什么获取了锁以后还要中断线程呢？
当中断线程被唤醒时，并不知道被唤醒的原因，可能是当前线程在等待中被中断，也可能是释放了锁以后被唤醒。因此我们通过Thread.interrupted()方法检查中断标记（该方法返回了当前线程的中断状态，并将当前线程的中断标识设置为False），并记录下来，如果发现该线程被中断过，就再中断一次。
线程在等待资源的过程中被唤醒，唤醒后还是会不断地去尝试获取锁，直到抢到锁为止。也就是说，在整个流程中，并不响应中断，只是记录中断记录。最后抢到锁返回了，那么如果被中断过的话，就需要补充一次中断。