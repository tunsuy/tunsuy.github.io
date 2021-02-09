# 一文搞定ThreadLocal原理

## ThreadLocal是什么
ThreadLocal是一个关于创建线程局部变量的类。
通常情况下，我们创建的变量是可以被任何一个线程访问并修改的。而使用ThreadLocal创建的变量只能被当前线程访问，其他线程则无法访问和修改。

ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。
这里的变量是指的ThreadLocal泛型中的类型实例，比如 `ThreadLocal< StringBuilder >`，这里的变量就是指的StringBuilder实例。

那么哪些情况下需要创建这样一个线程局部变量呢？
最常见的ThreadLocal使用场景为 用来解决 数据库连接、Session管理等。为什么这些场景需要用到这个ThreadLocal呢？

下面我们以数据库连接为例，比如有这样一个类：
```java
class ConnectionManager {
     
    private static Connection connect = null;
     
    public static Connection getConnection() {
        if(connect == null){
            connect = DriverManager.getConnection();
        }
        return connect;
    }
     
    public static void closeConnection() {
        if(connect!=null)
            connect.close();
    }
}
```
这段代码在单线程中使用是没有任何问题的，但是如果在多线程中使用呢？很显然，在多线程中使用会存在线程安全问题：
* 第一，这里面的2个方法都没有进行同步，很可能在getConnection方法中会多次创建connect；
* 第二，由于connect是共享变量，那么必然在调用connect的地方需要使用到同步来保障线程安全，因为很可能一个线程在使用connect进行数据库操作，而另外一个线程调用closeConnection关闭链接。
  所以出于线程安全的考虑，必须将这段代码的两个方法进行同步处理，并且在调用connect的地方需要进行同步处理。但是这样将会大大影响程序执行效率，因为一个线程在使用connect进行数据库操作的时候，其他线程只有等待。

可能有的人的解决方法就是：既然不需要在线程之间共享这个变量，可以直接这样处理，在每个需要使用数据库连接的方法中具体使用时才创建数据库链接，然后在方法调用完毕再释放这个连接。
这样处理确实也没有任何问题，由于每次都是在方法内部创建的连接，那么线程之间自然不存在线程安全问题。但是这样会有一个致命的影响：导致服务器压力非常大，并且严重影响程序执行性能。由于在方法中需要频繁地开启和关闭数据库连接，这样不尽严重影响程序执行效率，还可能导致服务器压力巨大。
那么这种场景下就需要用到ThreadLocal，让这个connect在线程间隔离，同时在一个线程只会存在一个connect。

## ThreadLocal的实现
下面我们开始讲解ThreadLocal的具体是怎样实现上面说的效果的
ThreadLocal提供了类似于Map的set和get操作，下面逐一讲解。

### 1、Set值
set()方法代码如下：
```java
public void set(T value) {
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null)
		map.set(this, value);
	else
		createMap(t, value);
}
```
代码流程如下：
* 1、获取当前线程
* 2、根据当前线程，获取ThreadLocalMap类
* 3、如果该map存在，则直接更新当前线程值，
* 4、如果不存在，则创建该map

#### 1.1 ThreadLocalMap是什么？
ThreadLocalMap是ThreadLocal的一个静态内部类，它基本就是按照hashmap的语义实现了一个类似的hashmap：
```java
static class ThreadLocalMap {

	/**
	 * The entries in this hash map extend WeakReference, using
	 * its main ref field as the key (which is always a
	 * ThreadLocal object).  Note that null keys (i.e. entry.get()
	 * == null) mean that the key is no longer referenced, so the
	 * entry can be expunged from table.  Such entries are referred to
	 * as "stale entries" in the code that follows.
	 */
	static class Entry extends WeakReference<ThreadLocal<?>> {
		/** The value associated with this ThreadLocal. */
		Object value;

		Entry(ThreadLocal<?> k, Object v) {
			super(k);
			value = v;
		}
	}

	/**
	 * The initial capacity -- MUST be a power of two.
	 */
	private static final int INITIAL_CAPACITY = 16;

	/**
	 * The table, resized as necessary.
	 * table.length MUST always be a power of two.
	 */
	private Entry[] table;

	/**
	 * The number of entries in the table.
	 */
	private int size = 0;

	/**
	 * The next size value at which to resize.
	 */
	private int threshold; // Default to 0

	/**
	 * Set the resize threshold to maintain at worst a 2/3 load factor.
	 */
	private void setThreshold(int len) {
		threshold = len * 2 / 3;
	}

	/**
	 * Increment i modulo len.
	 */
	private static int nextIndex(int i, int len) {
		return ((i + 1 < len) ? i + 1 : 0);
	}

	/**
	 * Decrement i modulo len.
	 */
	private static int prevIndex(int i, int len) {
		return ((i - 1 >= 0) ? i - 1 : len - 1);
	}
	
	......
	
}
```
如果看到hashmap的源码实现的，就一定很熟悉，上面的代码实现跟hashmap很类似。
那么ThreadLocalMap里面的key对应的就是ThreadLocal<?>对象，value对应的就是ThreadLocal里面泛型类型的对象。也就是说业务要使用到的变量就是存储在这ThreadLocalMap中的。

#### 1.2 关联Thread
既然业务要使用到的变量是存储在ThreadLocalMap对象中的，那么怎么保证这个对象的线程安全性呢？我们在最开始说过，变量是要做到线程隔离，那么ThreadLocalMap对象一定也是存储在线程中，而不是全局的。
我们回到上面的set方法中，有这样一句：
```java
createMap(t, value);

void createMap(Thread t, T firstValue) {
	t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
可以看到，这里是初始化了一个ThreadLocalMap，然后将其关联在了当前线程，也就是说ThreadLocalMap是Thread的一个属性。


### 2、Get值
下面来看看get方法
```java
public T get() {
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null) {
		ThreadLocalMap.Entry e = map.getEntry(this);
		if (e != null) {
			@SuppressWarnings("unchecked")
			T result = (T)e.value;
			return result;
		}
	}
	return setInitialValue();
}
```
上面代码的流程如下：
* 1、获取当前线程；
* 2、根据当前线程获取到其ThreadLocalMap属性；
* 3、以ThreadLocal实例为key，从Map中获取出value并返回。

### 关于内存泄露
ThreadLocal在防止内存泄露方案做了几点努力，下面来说说。

#### 1、弱引用
因为ThreadLocal被ThreadLocalMap引用，而ThreadLocalMap又被Thread引用。因此在线程池这种情况下，某个线程创建了ThreadLocal，该线程的业务处理完之后，被放入了线程池，此时，如果ThreadLocal是强引用，就会导致ThreadLocal也会随着Thread一直存在，从而造成内存泄露。
因此，在ThreadLocalMap的实现中，对ThreadLocal采用了弱引用的方式：
```java
static class Entry extends WeakReference<ThreadLocal<?>> {
	/** The value associated with this ThreadLocal. */
	Object value;

	Entry(ThreadLocal<?> k, Object v) {
		super(k);
		value = v;
	}
}
```
对对象进行弱引用不会影响垃圾回收器回收该对象，即如果一个对象只有弱引用存在了，则下次GC将会回收掉该对象（不管当前内存空间足够与否）。

#### 2、自动清理
我们注意到Entry对象中，虽然Key(ThreadLocal)是通过弱引用引入的，但是value即变量值本身是通过强引用引入。

这就导致，假如不作任何处理，由于ThreadLocalMap和线程的生命周期是一致的，当线程资源长期不释放，即使ThreadLocal本身由于弱引用机制已经回收掉了，但value还是驻留在线程的ThreadLocalMap的Entry中。即存在key为null，但value却有值的无效Entry。导致内存泄漏。
因此ThreadLocal内部已经为我们做了一定的防止内存泄漏的工作。
```java
private int expungeStaleEntry(int staleSlot) {
	Entry[] tab = table;
	int len = tab.length;

	// expunge entry at staleSlot
	tab[staleSlot].value = null;
	tab[staleSlot] = null;
	size--;

	// Rehash until we encounter null
	Entry e;
	int i;
	for (i = nextIndex(staleSlot, len);
		 (e = tab[i]) != null;
		 i = nextIndex(i, len)) {
		ThreadLocal<?> k = e.get();
		if (k == null) {
			e.value = null;
			tab[i] = null;
			size--;
		} else {
			int h = k.threadLocalHashCode & (len - 1);
			if (h != i) {
				tab[i] = null;

				// Unlike Knuth 6.4 Algorithm R, we must scan until
				// null because multiple entries could have been stale.
				while (tab[h] != null)
					h = nextIndex(h, len);
				tab[h] = e;
			}
		}
	}
	return i;
}
```
上述方法的作用是擦除某个下标的Entry（置为null，可以回收），同时检测整个Entry[]表中对key为null的Entry一并擦除，重新调整索引。
该方法，在每次调用ThreadLocal的get、set、remove方法时都会执行，即ThreadLocal内部已经帮我们做了对key为null的Entry的清理工作。
但是该工作是有触发条件的，需要调用相应方法，假如我们使用完之后不做任何处理是不会触发的。