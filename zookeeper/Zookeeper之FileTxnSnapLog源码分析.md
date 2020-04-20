## FileTxnSnapLog源码分析

### 1. 内部类
FileTxnSnapLog包含了PlayBackListener内部类，用来接收事务应用过程中的回调，在Zookeeper数据恢复后期，会有事务修正过程，此过程会回调PlayBackListener来进行对应的数据修正。其源码如下　
```java
public interface PlayBackListener {

	void onTxnLoaded(TxnHeader hdr, Record rec, TxnDigest digest);

}
```
在完成事务操作后，会调用到onTxnLoaded方法进行相应的处理。

### 2. restore函数：
```java
public long restore(DataTree dt, Map<Long, Integer> sessions, PlayBackListener listener) throws IOException {
	long snapLoadingStartTime = Time.currentElapsedTime();
	long deserializeResult = snapLog.deserialize(dt, sessions);
	ServerMetrics.getMetrics().STARTUP_SNAP_LOAD_TIME.add(Time.currentElapsedTime() - snapLoadingStartTime);
	FileTxnLog txnLog = new FileTxnLog(dataDir);
	boolean trustEmptyDB;
	File initFile = new File(dataDir.getParent(), "initialize");
	if (Files.deleteIfExists(initFile.toPath())) {
		LOG.info("Initialize file found, an empty database will not block voting participation");
		trustEmptyDB = true;
	} else {
		trustEmptyDB = autoCreateDB;
	}

	RestoreFinalizer finalizer = () -> {
		long highestZxid = fastForwardFromEdits(dt, sessions, listener);
		// The snapshotZxidDigest will reset after replaying the txn of the
		// zxid in the snapshotZxidDigest, if it's not reset to null after
		// restoring, it means either there are not enough txns to cover that
		// zxid or that txn is missing
		DataTree.ZxidDigest snapshotZxidDigest = dt.getDigestFromLoadedSnapshot();
		if (snapshotZxidDigest != null) {
			LOG.warn(
					"Highest txn zxid 0x{} is not covering the snapshot digest zxid 0x{}, "
							+ "which might lead to inconsistent state",
					Long.toHexString(highestZxid),
					Long.toHexString(snapshotZxidDigest.getZxid()));
		}
		return highestZxid;
	};

	if (-1L == deserializeResult) {
		/* this means that we couldn't find any snapshot, so we need to
		 * initialize an empty database (reported in ZOOKEEPER-2325) */
		if (txnLog.getLastLoggedZxid() != -1) {
			// ZOOKEEPER-3056: provides an escape hatch for users upgrading
			// from old versions of zookeeper (3.4.x, pre 3.5.3).
			if (!trustEmptySnapshot) {
				throw new IOException(EMPTY_SNAPSHOT_WARNING + "Something is broken!");
			} else {
				LOG.warn("{}This should only be allowed during upgrading.", EMPTY_SNAPSHOT_WARNING);
				return finalizer.run();
			}
		}

		if (trustEmptyDB) {
			/* TODO: (br33d) we should either put a ConcurrentHashMap on restore()
			 *       or use Map on save() */
			save(dt, (ConcurrentHashMap<Long, Integer>) sessions, false);

			/* return a zxid of 0, since we know the database is empty */
			return 0L;
		} else {
			/* return a zxid of -1, since we are possibly missing data */
			LOG.warn("Unexpected empty data tree, setting zxid to -1");
			dt.lastProcessedZxid = -1L;
			return -1L;
		}
	}

	return finalizer.run();
}
```
说明：restore用于恢复datatree和sessions，其步骤大致如下
* 根据snapshot文件反序列化datatree和sessions
* 获取比snapshot文件中的zxid+1大的log文件的迭代器，以对log文件中的事务进行迭代
* 迭代log文件的每个事务，并且将该事务应用在datatree中，同时会调用onTxnLoaded函数进行后续处理
* 关闭迭代器，返回log文件中最后一个事务的zxid（作为最高的zxid）

### 2. save函数　
```java
public void save(
	DataTree dataTree,
	ConcurrentHashMap<Long, Integer> sessionsWithTimeouts,
	boolean syncSnap) throws IOException {
	long lastZxid = dataTree.lastProcessedZxid;
	File snapshotFile = new File(snapDir, Util.makeSnapshotName(lastZxid));
	LOG.info("Snapshotting: 0x{} to {}", Long.toHexString(lastZxid), snapshotFile);
	try {
		snapLog.serialize(dataTree, sessionsWithTimeouts, snapshotFile, syncSnap);
	} catch (IOException e) {
		if (snapshotFile.length() == 0) {
			/* This may be caused by a full disk. In such a case, the server
			 * will get stuck in a loop where it tries to write a snapshot
			 * out to disk, and ends up creating an empty file instead.
			 * Doing so will eventually result in valid snapshots being
			 * removed during cleanup. */
			if (snapshotFile.delete()) {
				LOG.info("Deleted empty snapshot file: {}", snapshotFile.getAbsolutePath());
			} else {
				LOG.warn("Could not delete empty snapshot file: {}", snapshotFile.getAbsolutePath());
			}
		} else {
			/* Something else went wrong when writing the snapshot out to
			 * disk. If this snapshot file is invalid, when restarting,
			 * ZooKeeper will skip it, and find the last known good snapshot
			 * instead. */
		}
		throw e;
	}
}
```
说明：save函数用于将sessions和datatree保存至snapshot文件中，其大致步骤如下
* 获取内存数据库中已经处理的最新的zxid
* 根据zxid和快照目录生成snapshot文件
* 将datatree（内存数据库）、sessionsWithTimeouts序列化至快照文件。