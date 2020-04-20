## FileSnap源码分析

FileSnap实现了SnapShot接口，主要用作存储、序列化、反序列化、访问相应snapshot文件。

### 1. deserialize函数
```java
public long deserialize(DataTree dt, Map<Long, Integer> sessions) throws IOException {
	// we run through 100 snapshots (not all of them)
	// if we cannot get it running within 100 snapshots
	// we should  give up
	List<File> snapList = findNValidSnapshots(100);
	if (snapList.size() == 0) {
		return -1L;
	}
	File snap = null;
	long snapZxid = -1;
	boolean foundValid = false;
	for (int i = 0, snapListSize = snapList.size(); i < snapListSize; i++) {
		snap = snapList.get(i);
		LOG.info("Reading snapshot {}", snap);
		snapZxid = Util.getZxidFromName(snap.getName(), SNAPSHOT_FILE_PREFIX);
		try (CheckedInputStream snapIS = SnapStream.getInputStream(snap)) {
			InputArchive ia = BinaryInputArchive.getArchive(snapIS);
			deserialize(dt, sessions, ia);
			SnapStream.checkSealIntegrity(snapIS, ia);

			// Digest feature was added after the CRC to make it backward
			// compatible, the older code can still read snapshots which
			// includes digest.
			//
			// To check the intact, after adding digest we added another
			// CRC check.
			if (dt.deserializeZxidDigest(ia, snapZxid)) {
				SnapStream.checkSealIntegrity(snapIS, ia);
			}

			foundValid = true;
			break;
		} catch (IOException e) {
			LOG.warn("problem reading snap file {}", snap, e);
		}
	}
	if (!foundValid) {
		throw new IOException("Not able to find valid snapshots in " + snapDir);
	}
	dt.lastProcessedZxid = snapZxid;
	lastSnapshotInfo = new SnapshotInfo(dt.lastProcessedZxid, snap.lastModified() / 1000);

	// compare the digest if this is not a fuzzy snapshot, we want to compare
	// and find inconsistent asap.
	if (dt.getDigestFromLoadedSnapshot() != null) {
		dt.compareSnapshotDigests(dt.lastProcessedZxid);
	}
	return dt.lastProcessedZxid;
}
```
说明：deserialize主要用作反序列化，并将反序列化结果保存至dt和sessions中。 其大致步骤如下
* 获取100个合法的snapshot文件，并且snapshot文件已经通过zxid进行降序排序
* 遍历100个snapshot文件，从zxid最大的开始，读取该文件，并创建相应的InputArchive
* 调用deserialize(dt,sessions, ia)函数完成反序列化操作
* 验证从文件中读取的Checksum是否与新生的Checksum相等，若不等，则抛出异常
* 跳出循环并关闭相应的输入流，并从文件名中解析出相应的zxid返回。
* 在遍历100个snapshot文件后仍然无法找到通过验证的文件，则抛出异常。

在deserialize函数中，会调用findNValidSnapshots以及同名的deserialize(dt,sessions, ia)函数。
findNValidSnapshots函数主要是查找N个合法的snapshot文件并进行降序排序后返回。
deserialize(dt,sessions, ia)函数主要作用反序列化，并将反序列化结果保存至header和sessions中。其中会验证header的魔数是否相等。

### 2. serialize函数
```java
public synchronized void serialize(
	DataTree dt,
	Map<Long, Integer> sessions,
	File snapShot,
	boolean fsync) throws IOException {
	if (!close) {
		try (CheckedOutputStream snapOS = SnapStream.getOutputStream(snapShot, fsync)) {
			OutputArchive oa = BinaryOutputArchive.getArchive(snapOS);
			FileHeader header = new FileHeader(SNAP_MAGIC, VERSION, dbId);
			serialize(dt, sessions, oa, header);
			SnapStream.sealStream(snapOS, oa);

			// Digest feature was added after the CRC to make it backward
			// compatible, the older code cal still read snapshots which
			// includes digest.
			//
			// To check the intact, after adding digest we added another
			// CRC check.
			if (dt.serializeZxidDigest(oa)) {
				SnapStream.sealStream(snapOS, oa);
			}

			lastSnapshotInfo = new SnapshotInfo(
				Util.getZxidFromName(snapShot.getName(), SNAPSHOT_FILE_PREFIX),
				snapShot.lastModified() / 1000);
		}
	}
}
```
该函数用于将header、sessions、dt序列化至本地snapshot文件中，并且在最后会写入"/"字符。该方法是同步的，即是线程安全的。
`FileHeader header = new FileHeader(SNAP_MAGIC, VERSION, dbId);`可以看出写入了magic、version、dbid，也就是在之前的文章中介绍的，snapshot文件解析之后的第一行。
另外还会序列号摘要