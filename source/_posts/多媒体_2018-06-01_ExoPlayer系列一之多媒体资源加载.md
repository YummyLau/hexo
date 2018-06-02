---
title: ExoPlayer系列一之多媒体加载
layout: post
date: 2018-06-01 20:16:10
comments: true
categories: 多媒体
tags: [多媒体,ExoPlayer,Android]
---

**ExoPlayer** 库提供了多途径加载多媒体资源 ， 如记载在线视频文件、 加载本地音频文件、 读取本地视频缓存等 。 其核心围绕 `DataSource` 类提供多种途径加载媒体的默认实现类 。 

### DataSource

`DataSource` 是可读数据流的核心接口 ， 暴露构建不同场景下读取数据流的 `Factory` 接口 。 如 `FileDataSource` 读取的是 **File** 文件数据 ， `HttpDataSource` 则是从 **HTTP** 协议中读取服务器数据 。

#### DefaultDataSource

`DefaultDataSource` 及 `DefaultDataSourceFactory` 是 **ExoPlayer** 库默认实现处理媒体加载的类 , 其实现内部逻辑为

* 根据给定 uri , 识别 Scheme 来判断获取数据流的方式
* 根据不同的方式 ， 构建不同的 *DataSource* 实例
* *DataSource* 实例代理 `DefaultDataSource#read` 及 `DefaultDataSource#open` 的实现

`DefaultDataSource` 的逻辑非常简单明了 ， 内部维护了真正读取数据的 `DataSource` 实例 。

#### #open

```
  /**
   * Opens the source to read the specified data.
   * <p>
   * Note: If an {@link IOException} is thrown, callers must still call {@link #close()} to ensure
   * that any partial effects of the invocation are cleaned up.
   *
   * @param dataSpec Defines the data to be read.
   * @throws IOException If an error occurs opening the source. {@link DataSourceException} can be
   *     thrown or used as a cause of the thrown exception to specify the reason of the error.
   * @return The number of bytes that can be read from the opened source. For unbounded requests
   *     (i.e. requests where {@link DataSpec#length} equals {@link C#LENGTH_UNSET}) this value
   *     is the resolved length of the request, or {@link C#LENGTH_UNSET} if the length is still
   *     unresolved. For all other requests, the value returned will be equal to the request's
   *     {@link DataSpec#length}.
   */
  long open(DataSpec dataSpec) throws IOException;
```

该方法用于 “ 打开数据源来读取特定的数据 ” 。 其中 `DataSpec` 定义了操作数据的信息 ， 包括是否压缩、 是否缓存、 请求源、 可读长度、 读取位置、 标志位等等 。

方法返回 `long` 数据源长度 。 下面以 `FileDataSource` 及 `DefaultHttpDataSource` 加载场景来进一步加深理解 （以下分析代码保留主逻辑略细节） 。

```
from FileDataSource.java

 @Override
  public long open(DataSpec dataSpec) throws FileDataSourceException {

    try {
      //1. 获取可读文件
      uri = dataSpec.uri;
      file = new RandomAccessFile(dataSpec.uri.getPath(), "r");
      file.seek(dataSpec.position);

      //2. 获取dataSpec定义长度 ， 若数据信息为未定义 ， 则默认从 position位置以后的长度
      bytesRemaining = dataSpec.length == C.LENGTH_UNSET ? file.length() - dataSpec.position
          : dataSpec.length;
      if (bytesRemaining < 0) {
        throw new EOFException();
      }
    } catch (IOException e) {
      throw new FileDataSourceException(e);
    }

    //...
    return bytesRemaining;
  }
```

上述读取文件的实现简单明了 ， 而通过 Http 请求则相对复杂些 ：

```
  @Override
  public long open(DataSpec dataSpec) throws HttpDataSourceException {
    this.dataSpec = dataSpec;
    this.bytesRead = 0;
    this.bytesSkipped = 0;

    //1. 建立http链接， 使用的HttpUrlConnection
    try {
      connection = makeConnection(dataSpec);
    } catch (IOException e) {
      throw new HttpDataSourceException("Unable to connect to " + dataSpec.uri.toString(), e,
          dataSpec, HttpDataSourceException.TYPE_OPEN);
    }

    //2. 获取请求结果
    int responseCode;
    try {
      responseCode = connection.getResponseCode();
    } catch (IOException e) {
      closeConnectionQuietly();
      throw new HttpDataSourceException("Unable to connect to " + dataSpec.uri.toString(), e,
          dataSpec, HttpDataSourceException.TYPE_OPEN);
    }

    //3. 判断是否请求正常
    if (responseCode < 200 || responseCode > 299) {
      ...
      throw exception;
    }

    //4. 判断数据是否合法可用
    String contentType = connection.getContentType();
    if (contentTypePredicate != null && !contentTypePredicate.evaluate(contentType)) {
      throw exception;
    }

    //5. 计算读取数据的长度
    bytesToSkip = responseCode == 200 && dataSpec.position != 0 ? dataSpec.position : 0;
    if (!dataSpec.isFlagSet(DataSpec.FLAG_ALLOW_GZIP)) {
      if (dataSpec.length != C.LENGTH_UNSET) {
        bytesToRead = dataSpec.length;
      } else {
        long contentLength = getContentLength(connection);
        bytesToRead = contentLength != C.LENGTH_UNSET ? (contentLength - bytesToSkip)
            : C.LENGTH_UNSET;
      }
    } else {
      //如果数据被压缩，实际上压缩后的数据并不是我们所要计算的数据。 但是此处暂时返回压缩后的数据长度
      bytesToRead = dataSpec.length;
    }

    //6. 获取 InputStream 写入流
    try {
      inputStream = connection.getInputStream();
    } catch (IOException e) {
      ...
      throw exception;
    }

    //7. 设置 source 已经打开并执行存在的回调。
    opened = true;
    if (listener != null) {
      listener.onTransferStart(this, dataSpec);
    }
    return bytesToRead;
  }

```

无论是 `FileDataSource` 还是 `DefaultHttpDataSource` ， `#open` 都保留了可读数据的入口 ， 分别是 *file* 和 *inputStream* ，为 `#read` 中做好了铺垫 。

#### #read

```
  /**
   * Reads up to {@code length} bytes of data and stores them into {@code buffer}, starting at
   * index {@code offset}.
   * <p>
   * If {@code length} is zero then 0 is returned. Otherwise, if no data is available because the
   * end of the opened range has been reached, then {@link C#RESULT_END_OF_INPUT} is returned.
   * Otherwise, the call will block until at least one byte of data has been read and the number of
   * bytes read is returned.
   *
   * @param buffer The buffer into which the read data should be stored.
   * @param offset The start offset into {@code buffer} at which data should be written.
   * @param readLength The maximum number of bytes to read.
   * @return The number of bytes read, or {@link C#RESULT_END_OF_INPUT} if no data is available
   *     because the end of the opened range has been reached.
   * @throws IOException If an error occurs reading from the source.
   */
  int read(byte[] buffer, int offset, int readLength) throws IOException;
```

该方法用于 “ 在 #open 方法暴露的数据入口中， 从 offset 位置读取长度为 readLength 的数据并保存在 buffer 缓存区中 ” ， 返回读取的长度 。

同样我们以 `FileDataSource` 及 `DefaultHttpDataSource` 为例来进一步加深理解 （以下分析代码保留主逻辑略细节） 。

```
  @Override
  public int read(byte[] buffer, int offset, int readLength) throws FileDataSourceException {

    //1. 如果可读数据为 0 ， 则返回 0
    if (readLength == 0) {
      return 0;

    //2. 如果 file 没有剩余可读数据 ， 则返回 -1 标识读到尽头
    } else if (bytesRemaining == 0) {
      return C.RESULT_END_OF_INPUT;

    //3. 每次取剩余长度与将读长度的小值进行读取 ， 成功读取后更新剩余长度
    } else {
      int bytesRead;
      try {
        bytesRead = file.read(buffer, offset, (int) Math.min(bytesRemaining, readLength));
      } catch (IOException e) {
        throw new FileDataSourceException(e);
      }

      if (bytesRead > 0) {
        bytesRemaining -= bytesRead;
        if (listener != null) {
          listener.onBytesTransferred(this, bytesRead);
        }
      }
      return bytesRead;
    }
  }
```

从上述方法可知 ， **File** 可读数据越读越小 ， 直到返回 `C.RESULT_END_OF_INPUT` 标志着文件已经读取完毕 。

```
  @Override
  public int read(byte[] buffer, int offset, int readLength) throws HttpDataSourceException {
    try {
      //1. 忽略所需要所需要跳过的字节
      skipInternal();

      //2. 真正读取数据的实现
      return readInternal(buffer, offset, readLength);
    } catch (IOException e) {
      throw new HttpDataSourceException(e, dataSpec, HttpDataSourceException.TYPE_READ);
    }
  }
```

上述 1 中为何会存在所需要跳过的字节数据呢 ？ 在 `#open` 实现中 ， 其实有一个代码段涉及到

```
    // If we requested a range starting from a non-zero position and received a 200 rather than a
    // 206, then the server does not support partial requests. We'll need to manually skip to the
    // requested position.
    bytesToSkip = responseCode == 200 && dataSpec.position != 0 ? dataSpec.position : 0;
```
从注释中可知 ， 如果我们请求的数据块不是从其实 0 位置开始且服务端不支持 HTTP 206 （常用于断点续传） ， 则服务端会返回 200 且返回的数据会从 0 位置开始 。这时候我们接收的数据和本地 buffer 区中的数据存在 [0 , rangPosition) 区间重叠 ， 故这部分需要剔除 。

`dataSpec.position != 0` 意味着本地请求并不是从 0 位置开始 ， 故 `responseCode == 200 && dataSpec.position != 0` 条件满足时我们需要丢去 [0 , rangPosition) 区间数据 。 看看 `#skipInternal` 是否如我们所想 。

```
  private void skipInternal() throws IOException {
    //1. 不需要跳过任何字节数据
    if (bytesSkipped == bytesToSkip) {
      return;
    }
    ...

    //2. 从inputStream中读取 “跳过长度” 的字节并从 0 位置写入 skipBuffer
    while (bytesSkipped != bytesToSkip) {
      int readLength = (int) Math.min(bytesToSkip - bytesSkipped, skipBuffer.length);
      int read = inputStream.read(skipBuffer, 0, readLength);
      if (Thread.interrupted()) {
        throw new InterruptedIOException();
      }
      if (read == -1) {
        throw new EOFException();
      }
      bytesSkipped += read;
      if (listener != null) {
        listener.onBytesTransferred(this, read);
      }
    }
  }
```

了解为何跳过且如何跳过后 ， 看看 `#readInternal` 是如何读取数据的 。

```
  private int readInternal(byte[] buffer, int offset, int readLength) throws IOException {
    //1. 如果读取数据长度为 0 ， 则返回 0
    if (readLength == 0) {
      return 0;
    }

    //2. bytesRead记录已读数据 ， 计算剩余可读数据真实可读数据
    if (bytesToRead != C.LENGTH_UNSET) {
      long bytesRemaining = bytesToRead - bytesRead;
      if (bytesRemaining == 0) {
        return C.RESULT_END_OF_INPUT;
      }
      readLength = (int) Math.min(readLength, bytesRemaining);
    }

    //3. 可读数据写入 buffer 区
    int read = inputStream.read(buffer, offset, readLength);
    if (read == -1) {
      if (bytesToRead != C.LENGTH_UNSET) {
        // End of stream reached having not read sufficient data.
        throw new EOFException();
      }
      return C.RESULT_END_OF_INPUT;
    }

    //4. 累计已写入的数据
    bytesRead += read;
    if (listener != null) {
      listener.onBytesTransferred(this, read);
    }
    return read;
  }
```

实际上 `#read` 流程和 `FileDataSource` 基本是一致的 。

#### #close
```
  /**
   * Closes the source.
   * <p>
   * Note: This method must be called even if the corresponding call to {@link #open(DataSpec)}
   * threw an {@link IOException}. See {@link #open(DataSpec)} for more details.
   *
   * @throws IOException If an error occurs closing the source.
   */
  void close() throws IOException;
```
同常规 javaStream 流处理一样。 *close()* 用于资源加载完毕或者加载过程出现异常时调用 。
对于 `FileDataSource` ， 处理逻辑就是关闭 *file* :  

```
  @Override
  public void close() throws FileDataSourceException {
    uri = null;
    try {

	  //1. 关闭 file
      if (file != null) {
        file.close();
      }
    } catch (IOException e) {
      throw new FileDataSourceException(e);
    } finally {
      file = null;

      //2. 如果打开过且存在监听，则需要回调 onTransferEnd， 这个方法后续会在带宽预测中讲到
      if (opened) {
        opened = false;
        if (listener != null) {
          listener.onTransferEnd(this);
        }
      }
    }
  }
```

对于 *DefaultHttpDataSource* ， 处理逻辑就是关闭 *inputStream* :  

```
 @Override
  public void close() throws HttpDataSourceException {
    try {

	  //1. 关闭 inputStream
      if (inputStream != null) {
        maybeTerminateInputStream(connection, bytesRemaining());
        try {
          inputStream.close();
        } catch (IOException e) {
          throw new HttpDataSourceException(e, dataSpec, HttpDataSourceException.TYPE_CLOSE);
        }
      }
    } finally {
      inputStream = null;
      closeConnectionQuietly();

 	  //2. 如果打开过且存在监听，则需要回调 onTransferEnd
      if (opened) {
        opened = false;
        if (listener != null) {
          listener.onTransferEnd(this);
        }
      }
    }
  }
```
#### 扩展数据获取

如果仔细看上述 `#open` ， `#read` 和 `#open` ， 相信对如何自定义数据获取流程有一定的理解 。  
官方提供了扩展 `Okhttp` 方式获取数据组件 `extension-okhttp` ， 内部实现实际上和 `DefaultHttpDataSource` 十分相识 。  
对于常规数据获取 ， 官方 `library-core` 实际上已经十分完善 ， 扩展方面更是提供了 `extension-okhttp` , `extension-rtmp` 等 。
业务不尽相同 ， 如果上述组件都不满足业务需求的话 ， 再考虑自行扩展组件即可 。