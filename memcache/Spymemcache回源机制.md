## Spymemcache回源机制

```java
public Iterator<MemcachedNode> getSequence(String k) {
	// Seven searches gives us a 1 in 2^7 chance of hitting the
	// same dead node all of the time.
	String hashFlagKey = getKeyByHashFlag(k);
	return new KetamaIterator(hashFlagKey, 7, getKetamaNodes(), hashAlg);
}
```
进入`KetamaIterator`类
```java
protected KetamaIterator(final String k, final int t,
						 TreeMap<Long, MemcachedNode> ketamaNodes, final HashAlgorithm hashAlg) {
	super();
	this.ketamaNodes = ketamaNodes;
	this.hashAlg = hashAlg;
	hashVal = hashAlg.hash(k);
	remainingTries = t;
	key = k;
}

```
从该类的构造方法可以看出，7表示重试的次数，也是是重试节点的个数。

再来看下这个方法：
```java
protected void addOperation(final String key, final Operation o) {
	MemcachedNode placeIn = null;
	MemcachedNode primary = locator.getPrimary(key);
	LOGGER.debug("primary:{}, isActive:{}", primary.getSocketAddress(), primary.isActive());

	if (primary.isActive() || failureMode == FailureMode.Retry) {
		placeIn = primary;
	} else if (failureMode == FailureMode.Cancel) {
		o.cancel();
	} else {
		Iterator<MemcachedNode> i = locator.getSequence(key);
		while (placeIn == null && i.hasNext()) {
			MemcachedNode n = i.next();
			LOGGER.debug("next:{}, active:{}", n.getSocketAddress(), n.isActive());
			if (n.isActive()) {
				placeIn = n;
			}
		}
		if (placeIn == null && strategy.equals(SelectNodeFailedStrategy.NEXT)) {
			placeIn = selectNextActiveNode();
		}
		if (placeIn == null) {
			placeIn = primary;
			LOGGER.warn("Could not redistribute to another node, "
					+ "retrying primary node for {}.", key);
		}
	}

	assert o.isCancelled() || placeIn != null : "No node found for key " + key;
	if (placeIn != null) {
		addOperation(placeIn, o);
	} else {
		assert o.isCancelled() : "No node found for " + key + " (and not "
				+ "immediately cancelled)";
	}
}
```
注意如下循环：
```java
while (placeIn == null && i.hasNext()) {
	MemcachedNode n = i.next();
	LOGGER.debug("next:{}, active:{}", n.getSocketAddress(), n.isActive());
	if (n.isActive()) {
		placeIn = n;
	}
}
```
可以看到，调用了`KetamaIterator`类的hasnext和next方法：
我们进入这两个方法：
```java
public boolean hasNext() {
	return remainingTries > 0;
}

public MemcachedNode next() {
	try {
		return getNodeForKey(hashVal);
	} finally {
		nextHash();
	}
}
```
再进入方法:
```java
private void nextHash() {
	// this.calculateHash(Integer.toString(tries)+key).hashCode();
	long tmpKey = hashAlg.hash((numTries++) + key);
	// This echos the implementation of Long.hashCode()
	hashVal += (int) (tmpKey ^ (tmpKey >>> 32));
	hashVal &= 0xffffffffL; /* truncate to 32-bits */
	remainingTries--;
}
```
该方法是计算下一个节点范围的hash

综上，我们可以知道，spymemcache默认是从key计算出的原始节点出发，顺序查找7个节点，如果7个节点都down掉，则会重新回到key计算出的原始节点。