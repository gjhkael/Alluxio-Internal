#Alluxio Block 存储
我们知道alluxio是基于内存的分布式文件系统，同样的alluxio也是基于block形式来管理数据的
，那么alluxio是如何存储数据？用什么存储数据？如何管理block的呢？

接下来，我们将通过常用的文件写的demo来讲解，数据是如何一步一步的封装成block，然后如何
写入worker的。至于master和worker之间的通信、心跳以及如何通过master查找文件，在后续的章节进行
分析。

##文件写 Demo
首先上来先上一个write文件的demo，让大家从抽象层API来体会一下文件的写入操作。
```java 
//alluxio.client.file.BaseFileSystem 或者通过ctrl ＋ N查找类
private void writeFile(FileSystem fileSystem)
        throws IOException, AlluxioException {
        ByteBuffer buf = ByteBuffer.allocate(NUMBERS * 4);
        buf.order(ByteOrder.nativeOrder());
        for (int k = 0; k < NUMBERS; k++) {
            buf.putInt(k);
        }
        LOG.debug("Writing data...");
        long startTimeMs = CommonUtils.getCurrentMs();
        if (fileSystem.exists(mFilePath)) {
            fileSystem.delete(mFilePath);
        }
        FileOutStream os = fileSystem.createFile(mFilePath, mWriteOptions);
        os.write(buf.array());LineageFileSystem
        os.close();
        System.out.println((FormatUtils.formatTimeTakenMs(startTimeMs, "writeFile to file " + mFilePath)));
    }

```
构造好FileSystem之后，就可以使用fileSystem来做相应的操作了，createFile这个API会先调用client向master发送createFile的操作
，master在接受到请求后向mInodes插入一个InodeFile对象，从而表明该文件创建成功，如果出现其它异常的话，会抛出相应的异常信息。下面具体来看源码：

##BaseFileSystem
首先来看FileSystem的createFile API，由于FileSystem是接口类，实现它的类只有一个BaseFileSystem，继承BaseFileSystem的是LineageFileSystem，但是LineageFileSystem
还是实验性的东西，不建议大家开启这个特性，[详情点击](http://www.alluxio.org/docs/master/en/Lineage-API.html)。所以目前alluxio client都是通过BaseFileSystem
来进行操作的。
```java
//alluxio.client.file.FileSystemMasterClient 或者通过ctrl ＋ N查找类
  public FileOutStream createFile(AlluxioURI path, CreateFileOptions options)
      throws FileAlreadyExistsException, InvalidPathException, IOException, AlluxioException {
    FileSystemMasterClient masterClient = mFileSystemContext.acquireMasterClient();
    try {
      masterClient.createFile(path, options);
      LOG.debug("Created file " + path.getPath());
    } finally {
      mFileSystemContext.releaseMasterClient(masterClient);
    }
    return new FileOutStream(path, options.toOutStreamOptions());
  }
```
如上API，createFile之前先通过masterClient向master发送createFile请求，而这个过程只是发生元数据的存储，实际数据并没有发生写过程，只有元数据写成功之后，
才会new 一个FileOutStream用来真正的写数据，如果有兴趣了解createFile这个RPC请求的可以参考第三篇RPC章节，这章详细的通过mkdir 这个命令来介绍了alluxio client
向master发送RPC请求的完整过程。这里直接略过。

##FileOutStream
通过path和outStreamOtions构造FileOutStream后，可以真正的通过write api来写数据了。

首先来看看outStreamOtions具有那些选项：
```java
//alluxio.client.file.options.OutStreamOptions 或者通过ctrl ＋ N查找类
public final class OutStreamOptions {
  private long mBlockSizeBytes;
  private long mTtl;
  private TtlAction mTtlAction;
  private FileWriteLocationPolicy mLocationPolicy;
  private WriteType mWriteType;
  private Permission mPermission;

  /**
   * @return the default {@link OutStreamOptions}
   */
  public static OutStreamOptions defaults() {
    return new OutStreamOptions();
  }

  private OutStreamOptions() {
    mBlockSizeBytes = Configuration.getBytes(PropertyKey.USER_BLOCK_SIZE_BYTES_DEFAULT);
    mTtl = Constants.NO_TTL;
    mTtlAction = TtlAction.DELETE;

    try {
      mLocationPolicy = CommonUtils.createNewClassInstance(
          Configuration.<FileWriteLocationPolicy>getClass(
              PropertyKey.USER_FILE_WRITE_LOCATION_POLICY), new Class[] {}, new Object[] {});
    } catch (Exception e) {
      throw Throwables.propagate(e);
    }
    mWriteType = Configuration.getEnum(PropertyKey.USER_FILE_WRITE_TYPE_DEFAULT, WriteType.class);
    mPermission = Permission.defaults();
    try {
      // Set user and group from user login module, and apply default file UMask.
      mPermission.applyFileUMask().setOwnerFromLoginModule();
    } catch (IOException e) {
      // Fall through to system property approach
    }
  }
}
```
可以看到，Options包含了mBlockSizeBytes、mTtll、mTtlAction、mLocationPolicy、mWriteType、mPermission这么几个参数。

- blockSizeByte : block的大小，默认是512MB，可以通过alluxio.user.block.size.bytes.default参数来配置，这个属于client端设置。
- ttl ：文件或者文件夹的生命周期，超过这个生命周期，会被alluixo Master守护进程启动的定时任务清除掉。默认是－1s，表示永久保存。
- ttlAction：表示ttl所触发的action操作，目前有两种，delete和free，delete会将文件或者文件夹删除连同底层文件系统一起删除，free只会将数据从alluxio中删除，元数据和
底层文件系统中的文件还在。大家可以通过alluxio fs命令来设置ttl，也可以在createFileOption里面设置。
- locationPolicy：文件写时，block位置选择策略，默认是LocalFirstPolicy，可以通过alluxio.user.file.write.location.policy.class参数进行配置。
- writeType：文件写类型，详情参考第一章节，包含多种写类型。可以通过alluxio.user.file.writetype.default参数设置。
- permission：文件权限设置，可以通过alluxio.security.authorization.permission.umask来控制，默认是777。

ok，现在回到正题，看看outputstream如何write数据：

```java
//文件位置 alluxio.client.file.FileOutStream，或者通过ctrl ＋ N查找类
  @Override
  public void write(byte[] b) throws IOException {
    Preconditions.checkArgument(b != null, PreconditionMessage.ERR_WRITE_BUFFER_NULL);
    write(b, 0, b.length);
  }

  @Override
  public void write(byte[] b, int off, int len) throws IOException {
    Preconditions.checkArgument(b != null, PreconditionMessage.ERR_WRITE_BUFFER_NULL);
    Preconditions.checkArgument(off >= 0 && len >= 0 && len + off <= b.length,
        PreconditionMessage.ERR_BUFFER_STATE.toString(), b.length, off, len);

    if (mShouldCacheCurrentBlock) {
      try {
        int tLen = len;
        int tOff = off;
        while (tLen > 0) {
          if (mCurrentBlockOutStream == null || mCurrentBlockOutStream.remaining() == 0) {
            getNextBlock();
          }
          long currentBlockLeftBytes = mCurrentBlockOutStream.remaining();
          if (currentBlockLeftBytes >= tLen) {
            mCurrentBlockOutStream.write(b, tOff, tLen);
            tLen = 0;
          } else {
            mCurrentBlockOutStream.write(b, tOff, (int) currentBlockLeftBytes);
            tOff += currentBlockLeftBytes;
            tLen -= currentBlockLeftBytes;
          }
        }
      } catch (IOException e) {
        handleCacheWriteException(e);
      }
    }

    if (mUnderStorageType.isSyncPersist()) {
      mUnderStorageOutputStream.write(b, off, len);
      Metrics.BYTES_WRITTEN_UFS.inc(len);
    }
    mBytesWritten += len;
  }
```
可以看到write一个byte[]的时候，通过一些条件检查之后，首先调用getNextBlock()方法，然后调用mCurrentBlockOutStream来真正的将数据写到相应的worker中去的。

```java
  //文件位置 alluxio.client.file.FileOutStream，或者通过ctrl ＋ N查找类
  private void getNextBlock() throws IOException {
    if (mCurrentBlockOutStream != null) {
      Preconditions.checkState(mCurrentBlockOutStream.remaining() <= 0,
          PreconditionMessage.ERR_BLOCK_REMAINING);
      mPreviousBlockOutStreams.add(mCurrentBlockOutStream);
    }

    if (mAlluxioStorageType.isStore()) {
      mCurrentBlockOutStream =
          mContext.getAlluxioBlockStore().getOutStream(getNextBlockId(), mBlockSize, mOptions);
      mShouldCacheCurrentBlock = true;
    }
  }
```
look,mCurrentBlockOutStream是通过mContext先get到AlluxioBlockStore，然后再get到outStream。
```java
//alluxio.client.block.AlluxioBlockStore 或者通过ctrl ＋ N查找类
  public BufferedBlockOutStream getOutStream(long blockId, long blockSize, OutStreamOptions options)
      throws IOException {
    WorkerNetAddress address;
    FileWriteLocationPolicy locationPolicy = Preconditions.checkNotNull(options.getLocationPolicy(),
        PreconditionMessage.FILE_WRITE_LOCATION_POLICY_UNSPECIFIED);
    try {
      address = locationPolicy.getWorkerForNextBlock(getWorkerInfoList(), blockSize);
    } catch (AlluxioException e) {
      throw new IOException(e);
    }
    return getOutStream(blockId, blockSize, address);
  }

```
可以看到，这边根据locationPolicy会根据用户设置的文件写位置选择策略来返回worker的IP地址，将blockId，blockSize和address再调用getOutStream
从而返回具体的outStream。
```java
 public BufferedBlockOutStream getOutStream(long blockId, long blockSize, WorkerNetAddress address)
      throws IOException {
    if (blockSize == -1) {
      try (CloseableResource<BlockMasterClient> blockMasterClientResource =
          mContext.acquireMasterClientResource()) {
        blockSize = blockMasterClientResource.get().getBlockInfo(blockId).getLength();
      } catch (AlluxioException e) {
        throw new IOException(e);
      }
    }
    // No specified location to write to.
    if (address == null) {
      throw new RuntimeException(ExceptionMessage.NO_WORKER_AVAILABLE.getMessage());
    }
    // Location is local.
    if (mLocalHostName.equals(address.getHost())) {
      return new LocalBlockOutStream(blockId, blockSize, address, mContext);
    }
    // Location is specified and it is remote.
    return new RemoteBlockOutStream(blockId, blockSize, address, mContext);
  }
```
look,这边会返回一个BufferedBlockOutStream，如果当前节点的hostname和get过来的address的hostname相同，则返回LocalBlockOutStream，否则
返回RemoteBlockOutStream。

接下来重点讲讲RemoteBlockOutStream，因为RemoteBlockOutStream继承了BufferedBlockOutStream，所以接下里结合BufferedBlockOutStream进行源码的分析

##BufferedBlockOutStream
FileOutputStream.write将byte[]写入alluxio最终调用的是RemoteBlockOutStream的write方法。所以先看write
```java
//alluxio.client.block.BufferedBlockOutStream
public void write(byte[] b, int off, int len) throws IOException {
    if (len == 0) {
      return;
    }

    // Write the non-empty buffer if the new write will overflow it.
    if (mBuffer.position() > 0 && mBuffer.position() + len > mBuffer.limit()) {
      flush();
    }

    // If this write is larger than half of buffer limit, then write it out directly
    // to the remote block. Before committing the new writes, need to make sure
    // all bytes in the buffer are written out first, to prevent out-of-order writes.
    // Otherwise, when the write is small, write the data to the buffer.
    if (len > mBuffer.limit() / 2) {
      if (mBuffer.position() > 0) {
        flush();
      }
      unBufferedWrite(b, off, len);
    } else {
      mBuffer.put(b, off, len);
    }

    mWrittenBytes += len;
  }
```
- 如果byte[]写入的len不大于mBuffer.limit()/2的话，直接将byte数组写入到mBuffer，mBuffer是ByteBuffer数据类型，默认值为1MB，可以通过
alluxio.user.file.buffer.bytes参数进行设置。
- 如果大于该值的话，就会触发flush()方法，该方法已经被RemoteBlockOutStream重写，接下来调用writeToRemoteBlock将数据写入指定的block中，
同时将mBuffer.clear();清空buffer。

```java
  private void writeToRemoteBlock(byte[] b, int off, int len) throws IOException {
    mRemoteWriter.write(b, off, len);
    mFlushedBytes += len;
    Metrics.BYTES_WRITTEN_REMOTE.inc(len);
  }
```
最终通过mRemoteWriter来写，
```
  public RemoteBlockOutStream(long blockId,
      long blockSize,
      WorkerNetAddress address,
      BlockStoreContext blockStoreContext) throws IOException {
    super(blockId, blockSize, blockStoreContext);
    mCloser = Closer.create();
    try {
      mRemoteWriter = mCloser.register(RemoteBlockWriter.Factory.create());
      mBlockWorkerClient = mCloser.register(mContext.createWorkerClient(address));

      mRemoteWriter.open(mBlockWorkerClient.getDataServerAddress(), mBlockId,
          mBlockWorkerClient.getSessionId());
    } catch (IOException e) {
      mCloser.close();
      throw e;
    }
  }
```
可以看到之前通过locationpolicy获得的address会传递给fileSystemConext来构造WorkerClient。同时RemoteWriter会open这个client。
mRemoteWriter默认是NettyRemoteBlockWriter，可以通过alluxio.user.block.remote.writer.class来设置。

看看NettyRemoteBlockWriter如何写数据的

##NettyRemoteBlockWriter
```java
public final class NettyRemoteBlockWriter implements RemoteBlockWriter {
  private static final Logger LOG = LoggerFactory.getLogger(Constants.LOGGER_TYPE);

  private final Callable<Bootstrap> mClientBootstrap;

  private boolean mOpen;
  private InetSocketAddress mAddress;
  private long mBlockId;
  private long mSessionId;

  // Total number of bytes written to the remote block.
  private long mWrittenBytes;

  /**
   * Creates a new {@link NettyRemoteBlockWriter}.
   */
  public NettyRemoteBlockWriter() {
    mClientBootstrap = NettyClient.bootstrapBuilder();
    mOpen = false;
  }

  @Override
  public void open(InetSocketAddress address, long blockId, long sessionId) throws IOException {
    if (mOpen) {
      throw new IOException(
          ExceptionMessage.WRITER_ALREADY_OPEN.getMessage(mAddress, mBlockId, mSessionId));
    }
    mAddress = address;
    mBlockId = blockId;
    mSessionId = sessionId;
    mWrittenBytes = 0;
    mOpen = true;
  }

  @Override
  public void close() {
    if (mOpen) {
      mOpen = false;
    }
  }

  @Override
  public void write(byte[] bytes, int offset, int length) throws IOException {
    SingleResponseListener listener = null;
    Channel channel = null;
    Metrics.NETTY_BLOCK_WRITE_OPS.inc();
    try {
      channel = BlockStoreContext.acquireNettyChannel(mAddress, mClientBootstrap);
      listener = new SingleResponseListener();
      channel.pipeline().get(ClientHandler.class).addListener(listener);
      ChannelFuture channelFuture = channel.writeAndFlush(
          new RPCBlockWriteRequest(mSessionId, mBlockId, mWrittenBytes, length,
              new DataByteArrayChannel(bytes, offset, length))).sync();
      if (channelFuture.isDone() && !channelFuture.isSuccess()) {
        LOG.error("Failed to write to %s for block %d with error %s.", mAddress.toString(),
            mBlockId, channelFuture.cause());
        throw new IOException(channelFuture.cause());
      }

      RPCResponse response = listener.get(NettyClient.TIMEOUT_MS, TimeUnit.MILLISECONDS);

      switch (response.getType()) {
        case RPC_BLOCK_WRITE_RESPONSE:
          RPCBlockWriteResponse resp = (RPCBlockWriteResponse) response;
          RPCResponse.Status status = resp.getStatus();
          LOG.debug("status: {} from remote machine {} received", status, mAddress);

          if (status != RPCResponse.Status.SUCCESS) {
            throw new IOException(ExceptionMessage.BLOCK_WRITE_ERROR.getMessage(mBlockId,
                mSessionId, mAddress, status.getMessage()));
          }
          mWrittenBytes += length;
          break;
        case RPC_ERROR_RESPONSE:
          RPCErrorResponse error = (RPCErrorResponse) response;
          throw new IOException(error.getStatus().getMessage());
        default:
          throw new IOException(ExceptionMessage.UNEXPECTED_RPC_RESPONSE
              .getMessage(response.getType(), RPCMessage.Type.RPC_BLOCK_WRITE_RESPONSE));
      }
    } catch (Exception e) {
      Metrics.NETTY_BLOCK_WRITE_FAILURES.inc();
      try {
        // TODO(peis): We should not close the channel unless it is an exception caused by network.
        if (channel != null) {
          channel.close().sync();
        }
      } catch (InterruptedException ee) {
        Throwables.propagate(ee);
      }
      throw new IOException(e);
    } finally {
      if (channel != null && listener != null && channel.isActive()) {
        channel.pipeline().get(ClientHandler.class).removeListener(listener);
      }
      if (channel != null) {
        BlockStoreContext.releaseNettyChannel(mAddress, channel);
      }
    }
  }
}
```
通过下面代码写数据：
```java
      ChannelFuture channelFuture = channel.writeAndFlush(
          new RPCBlockWriteRequest(mSessionId, mBlockId, mWrittenBytes, length,
              new DataByteArrayChannel(bytes, offset, length))).sync();
```

###worker BlockWriter
如上所述，client通过Netty client向woker请求数据写，那么worker那边肯定会有相应的service handler来处理这个请求，找到BlockDataServerHandler类，在worker package中。

```java 
// alluxio.worker.netty.BlockDataServerHandler
 void handleBlockWriteRequest(final ChannelHandlerContext ctx, final RPCBlockWriteRequest req)
      throws IOException {
    final long sessionId = req.getSessionId();
    final long blockId = req.getBlockId();
    final long offset = req.getOffset();
    final long length = req.getLength();
    final DataBuffer data = req.getPayloadDataBuffer();

    BlockWriter writer = null;
    try {
      req.validate();
      ByteBuffer buffer = data.getReadOnlyByteBuffer();

      if (offset == 0) {
        // This is the first write to the block, so create the temp block file. The file will only
        // be created if the first write starts at offset 0. This allocates enough space for the
        // write.
        mWorker.createBlockRemote(sessionId, blockId, mStorageTierAssoc.getAlias(0), length);
      } else {
        // Allocate enough space in the existing temporary block for the write.
        mWorker.requestSpace(sessionId, blockId, length);
      }
      writer = mWorker.getTempBlockWriterRemote(sessionId, blockId);
      writer.append(buffer);

      Metrics.BYTES_WRITTEN_REMOTE.inc(data.getLength());
      RPCBlockWriteResponse resp =
          new RPCBlockWriteResponse(sessionId, blockId, offset, length, RPCResponse.Status.SUCCESS);
      ChannelFuture future = ctx.writeAndFlush(resp);
      future.addListener(new ClosableResourceChannelListener(writer));
    } catch (Exception e) {
      LOG.error("Error writing remote block : {}", e.getMessage(), e);
      RPCBlockWriteResponse resp =
          RPCBlockWriteResponse.createErrorResponse(req, RPCResponse.Status.WRITE_ERROR);
      ChannelFuture future = ctx.writeAndFlush(resp);
      future.addListener(ChannelFutureListener.CLOSE);
      if (writer != null) {
        writer.close();
      }
    }
  }
```
接下来，在DefaultBlockWrite中，调用 getTempBlockWriterRemote,返回BlockWriter
```
//alluxio.worker.block.DefaultBlockWorker
public BlockWriter getTempBlockWriterRemote(long sessionId, long blockId)
      throws BlockDoesNotExistException, IOException {
    return mBlockStore.getBlockWriter(sessionId, blockId);
}
 
```
mBlockStore是TieredBlockStore，为多级存储block存储管理器。返回LocalFileBlockWriter。
```
//alluxio.worker.block.TieredBlockStore
  public BlockWriter getBlockWriter(long sessionId, long blockId)
      throws BlockDoesNotExistException, IOException {
    // NOTE: a temp block is supposed to only be visible by its own writer, unnecessary to acquire
    // block lock here since no sharing
    // TODO(bin): Handle the case where multiple writers compete for the same block.
    try (LockResource r = new LockResource(mMetadataReadLock)) {
      TempBlockMeta tempBlockMeta = mMetaManager.getTempBlockMeta(blockId);
      return new LocalFileBlockWriter(tempBlockMeta.getPath());
    }
  }
```

localFileBlockWrite 调用write方法将数据存储起来。

```
//alluxio.worker.block.io.LocalFileBlockWriter
  private long write(long offset, ByteBuffer inputBuf) throws IOException {
    int inputBufLength = inputBuf.limit() - inputBuf.position();
    MappedByteBuffer outputBuf =
        mLocalFileChannel.map(FileChannel.MapMode.READ_WRITE, offset, inputBufLength);
    outputBuf.put(inputBuf);
    int bytesWritten = outputBuf.limit();
    BufferUtils.cleanDirectBuffer(outputBuf);
    return bytesWritten;
  }

```
可以看到LocalFileBlockWriter将根据inputBuf的长度然后申请相应的资源，其中mLocalFileChannel.map方法调用的是jdk的FileChannelImpl的map方法，返回的是MappedByteBuffer，也即是DirectByteBuffer,
DirectByteBuffer通过java的unsafe方法来直接申请系统内存，也就是堆外内存。




































