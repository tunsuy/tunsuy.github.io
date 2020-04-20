## FileTxnLog源码分析

### 1. append函数
```java
public synchronized boolean append(TxnHeader hdr, Record txn, TxnDigest digest) throws IOException {
	if (hdr == null) {
		return false;
	}
	if (hdr.getZxid() <= lastZxidSeen) {
		LOG.warn(
			"Current zxid {} is <= {} for {}",
			hdr.getZxid(),
			lastZxidSeen,
			hdr.getType());
	} else {
		lastZxidSeen = hdr.getZxid();
	}
	if (logStream == null) {
		LOG.info("Creating new log file: {}", Util.makeLogName(hdr.getZxid()));

		logFileWrite = new File(logDir, Util.makeLogName(hdr.getZxid()));
		fos = new FileOutputStream(logFileWrite);
		logStream = new BufferedOutputStream(fos);
		oa = BinaryOutputArchive.getArchive(logStream);
		FileHeader fhdr = new FileHeader(TXNLOG_MAGIC, VERSION, dbId);
		fhdr.serialize(oa, "fileheader");
		// Make sure that the magic number is written before padding.
		logStream.flush();
		filePadding.setCurrentSize(fos.getChannel().position());
		streamsToFlush.add(fos);
	}
	filePadding.padFile(fos.getChannel());
	byte[] buf = Util.marshallTxnEntry(hdr, txn, digest);
	if (buf == null || buf.length == 0) {
		throw new IOException("Faulty serialization for header " + "and txn");
	}
	Checksum crc = makeChecksumAlgorithm();
	crc.update(buf, 0, buf.length);
	oa.writeLong(crc.getValue(), "txnEntryCRC");
	Util.writeTxnBytes(oa, buf);

	return true;
}
```
说明：append函数主要用做向事务日志中添加一个条目，其大体步骤如下
* 检查TxnHeader是否为空，若不为空
* 检查logStream是否为空(初始化为空)
* 初始化写数据相关的流和FileHeader，并序列化FileHeader至指定文件
* 强制刷新（保证数据存到磁盘），并获取当前写入数据的大小
* 将事务头、事务和摘要序列化成ByteBuffer（使用Util.marshallTxnEntry函数）
* 使用Checksum算法更新ByteBuffer
* 将更新的ByteBuffer写入磁盘文件，返回true

代码中`filePadding.padFile(fos.getChannel());`可以看到调用了该方法，代码如下：
```java
long padFile(FileChannel fileChannel) throws IOException {
	long newFileSize = calculateFileSizeWithPadding(fileChannel.position(), currentSize, preAllocSize);
	if (currentSize != newFileSize) {
		fileChannel.write((ByteBuffer) fill.position(0), newFileSize - fill.remaining());
		currentSize = newFileSize;
	}
	return currentSize;
}
```
其主要作用是当文件大小不满64MB时，向文件填充0以达到64MB大小。

### 2. getLogFiles函数　
```java
public static File[] getLogFiles(File[] logDirList, long snapshotZxid) {
	List<File> files = Util.sortDataDir(logDirList, LOG_FILE_PREFIX, true);
	long logZxid = 0;
	// Find the log file that starts before or at the same time as the
	// zxid of the snapshot
	for (File f : files) {
		long fzxid = Util.getZxidFromName(f.getName(), LOG_FILE_PREFIX);
		if (fzxid > snapshotZxid) {
			break;
		}
		// the files
		// are sorted with zxid's
		if (fzxid > logZxid) {
			logZxid = fzxid;
		}
	}
	List<File> v = new ArrayList<File>(5);
	for (File f : files) {
		long fzxid = Util.getZxidFromName(f.getName(), LOG_FILE_PREFIX);
		if (fzxid < logZxid) {
			continue;
		}
		v.add(f);
	}
	return v.toArray(new File[0]);

}
```
说明：该函数的作用是找出刚刚小于或者等于snapshot的所有log文件。其步骤大致如下。
* 对所有log文件按照zxid进行升序排序
* 遍历所有log文件并记录刚刚小于或等于给定snapshotZxid的log文件的logZxid
* 再次遍历log文件，添加zxid大于等于logZxid的所有log文件

### 3. getLastLoggedZxid函数
```java
public long getLastLoggedZxid() {
	File[] files = getLogFiles(logDir.listFiles(), 0);
	long maxLog = files.length > 0 ? Util.getZxidFromName(files[files.length - 1].getName(), LOG_FILE_PREFIX) : -1;

	// if a log file is more recent we must scan it to find
	// the highest zxid
	long zxid = maxLog;
	TxnIterator itr = null;
	try {
		FileTxnLog txn = new FileTxnLog(logDir);
		itr = txn.read(maxLog);
		while (true) {
			if (!itr.next()) {
				break;
			}
			TxnHeader hdr = itr.getHeader();
			zxid = hdr.getZxid();
		}
	} catch (IOException e) {
		LOG.warn("Unexpected exception", e);
	} finally {
		close(itr);
	}
	return zxid;
}
```
说明：该函数主要用于获取记录在内存log中的最后一个zxid。其步骤大致如下
* 获取已排好序的所有log文件，并从最后一个文件中取出zxid作为候选的最大zxid
* 新生成FileTxnLog并读取内存中在zxid之后的所有事务
* 遍历所有事务并提取出相应的zxid，最后返回。

### 4. commit函数
```java
public synchronized void commit() throws IOException {
	if (logStream != null) {
		logStream.flush();
	}
	for (FileOutputStream log : streamsToFlush) {
		log.flush();
		if (forceSync) {
			long startSyncNS = System.nanoTime();

			FileChannel channel = log.getChannel();
			channel.force(false);

			syncElapsedMS = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startSyncNS);
			if (syncElapsedMS > fsyncWarningThresholdMS) {
				if (serverStats != null) {
					serverStats.incrementFsyncThresholdExceedCount();
				}

				LOG.warn(
					"fsync-ing the write ahead log in {} took {}ms which will adversely effect operation latency."
						+ "File size is {} bytes. See the ZooKeeper troubleshooting guide",
					Thread.currentThread().getName(),
					syncElapsedMS,
					channel.size());
			}

			ServerMetrics.getMetrics().FSYNC_TIME.add(syncElapsedMS);
		}
	}
	while (streamsToFlush.size() > 1) {
		streamsToFlush.poll().close();
	}

	// Roll the log file if we exceed the size limit
	if (txnLogSizeLimit > 0) {
		long logSize = getCurrentLogSize();

		if (logSize > txnLogSizeLimit) {
			LOG.debug("Log size limit reached: {}", logSize);
			rollLog();
		}
	}
}
```
说明：该函数主要用于提交事务日志至磁盘，其大致步骤如下
* 若日志流logStream不为空，则强制刷新至磁盘
* 遍历需要刷新至磁盘的所有流streamsToFlush并进行刷新
* 判断是否需要强制性同步，如是，则计算每个流的流式时间并在控制台给出警告
* 移除所有流并关闭。