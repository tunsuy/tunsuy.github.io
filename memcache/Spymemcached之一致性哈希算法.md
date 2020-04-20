## spymemcached框架之一致性哈希算法实现

文件：KetamaNodeLocator.java

1、初始化一致性哈希环
```java
protected void setKetamaNodes(List<MemcachedNode> nodes) {
	TreeMap<Long, MemcachedNode> newNodeMap =
			new TreeMap<Long, MemcachedNode>();
	int numReps = config.getNodeRepetitions();
	for (MemcachedNode node : nodes) {
		LOGGER.debug("node:{}", node);
		// Ketama does some special work with md5 where it reuses chunks.
		updateKetamaNode(newNodeMap, numReps, node, true);
	}
	assert newNodeMap.size() == numReps * nodes.size();
	ketamaNodes = newNodeMap;
}
```
进入`updateKetamaNode`方法：
```java
private void updateKetamaNode(TreeMap<Long, MemcachedNode> newNodeMap, int numReps, MemcachedNode node, boolean isAdd) {
	if (hashAlg == DefaultHashAlgorithm.KETAMA_HASH) {
		for (int i = 0; i < numReps / 4; i++) {
			byte[] digest =
					DefaultHashAlgorithm.computeMd5(config.getKeyForNode(node, i));
			LOGGER.debug("node:{}, i:{}, digest:{}",node.getSocketAddress(), i, digest);
			for (int h = 0; h < 4; h++) {
				Long k = ((long) (digest[3 + h * 4] & 0xFF) << 24)
						| ((long) (digest[2 + h * 4] & 0xFF) << 16)
						| ((long) (digest[1 + h * 4] & 0xFF) << 8)
						| (digest[h * 4] & 0xFF);
				if (isAdd) {
					newNodeMap.put(k, node);
					LOGGER.debug("addKetamaNode k:{}, node:{}", k, node.getSocketAddress());
				} else {
					newNodeMap.remove(k);
					LOGGER.debug("removeKetamaNode k:{}, node:{}", k, node.getSocketAddress());
				}

			}
		}
	} else {
		for (int i = 0; i < numReps; i++) {
			if (isAdd) {
				newNodeMap.put(hashAlg.hash(config.getKeyForNode(node, i)), node);
			} else {
				newNodeMap.remove(hashAlg.hash(config.getKeyForNode(node, i)));
			}

		}
	}

	//设置在ketaNode中的节点，使用synchronized避免
	synchronized (KETAMA_LOCK) {
		if (isAdd) {
			inKetaNodes.add(node.getSocketAddress());
		} else {
			inKetaNodes.remove(node.getSocketAddress());
		}
	}
	LOGGER.debug("inKetaNodes :{} .", inKetaNodes);

}
```
对上面代码解读如下：
`getNodeRepetitions()`方法负责读取配置信息，返回一个真实的Memcached节点对应的虚拟节点数，
默认情况下返回160，也就是说一个Memcached节点在一致性哈希环上对应有160个虚拟节点。

`getKeyForNode()`根据传进去的MemcacheNode对象和虚拟节点索引生成key值，返回值示例：“127.0.0.1:11311-0”
`computeMd5()`根据key生成16位的MD5摘要， 因此digest数组共16位

下面这段代码很关键：
```java
Long k = ((long) (digest[3 + h * 4] & 0xFF) << 24)
		| ((long) (digest[2 + h * 4] & 0xFF) << 16)
		| ((long) (digest[1 + h * 4] & 0xFF) << 8)
		| (digest[h * 4] & 0xFF);
```
结合`for (int i = 0; i < numReps / 4; i++) {...}`，我们知道：
将digest数组按每四位一组，通过位操作产生一个最大32位的长整数。之所以是32位是因为一致性哈希环取值范围为0～2^32; 回到上面的例子，对于一个Memcached节点譬如“127.0.0.1:11311”, 将通过for循环产生“127.0.0.1:11311-0”，“127.0.0.1:11311-1”… “127.0.0.1:11311-39”共40个副本，对于每个副本譬如“127.0.0.1:11311-0”, 将会产生4个长整数，对应一致性哈希环上的4个位置，所以默认配置的情况下，一个Memcached节点将在一致性哈希环上占据4×40=160个位置。

以k为key将MemcacheNode对象放到TreeMap里：
```java
newNodeMap.put(k, node);
```
由于TreeMap中的value是按Key排序的，因此可以通过TreeMap来模拟一致性哈希的环状结构，k值小的排在前，k值大的排在后。


2、查询
```java
public MemcachedNode getPrimary(final String k) {
	String hashFlagKey = getKeyByHashFlag(k);
	MemcachedNode rv = getNodeForKey(hashAlg.hash(hashFlagKey));
	assert rv != null : "Found no node for key " + hashFlagKey;
	return rv;
}
```
进入`getNodeForKey`方法：
```java
MemcachedNode getNodeForKey(long hash) {
	final MemcachedNode rv;
	if (!ketamaNodes.containsKey(hash)) {
		// Java 1.6 adds a ceilingKey method, but I'm still stuck in 1.5
		// in a lot of places, so I'm doing this myself.
		SortedMap<Long, MemcachedNode> tailMap = getKetamaNodes().tailMap(hash);
		if (tailMap.isEmpty()) {
			hash = getKetamaNodes().firstKey();
		} else {
			hash = tailMap.firstKey();
		}
	}
	rv = getKetamaNodes().get(hash);
	return rv;
}
```
关键在于`SortedMap<Long, MemcachedNode> tailMap = getKetamaNodes().tailMap(hash);`这句, TreeMap的tailMap()方法会返回一个SortedMap对象tailMap, tailMap中包含的所有key值都比传参hash大，这个操作相当于给定一个hash值，从一致性哈希环中按顺时针顺序查找节点，直到查找到第一个key值比传参hash大的节点，该节点就是该hash值所对应的Memcached节点。