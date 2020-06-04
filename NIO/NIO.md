# 一、初入NIO

## 1）是什么？

Java NIO是一个可以替代传统IO的一个IO API，它是从jdk1.4开始引入的，NIO提供了与传统IO不同的工作方式

## 2）什么不同？

传统的IO是通过读取到内存，然后在jvm中开辟一个堆内存空间，再通过复制/映射，读的时候从内存映射到jvm中，写的时候jvm映射到内存中，并且这个过程是独占锁去操作，是一个完全阻塞对的过程。

**NIO不同在于引入了以下三个概念：**

### a）Channels 和 Buffers（通道和缓存）

标准IO是基于字节流和字符流进行操作的，而NIO是基于通道和缓存进行操作的，数据是从通道读到缓冲区，缓冲区写到通道。

### b）Non-blocking IO（非阻塞）

NIO可以让你非阻塞式的使用IO；其实可以理解为让线程从通道读到缓冲区的时候，线程还可以做其他事情，线程从缓冲区写到通道的时候，线程也可以做其他事情。

### c）Selectors（选择器）

NIO引入选择器这个概念，是为了监听多个通道的事件，比如：数据到达、连接打开等等；所以单个线程是可以监听多个通道的数据。

# 二、概述

NIO虽然有很多类和组件，但是核心却是由Channel、Buffer和Selectors组成。

## 1）Channel和Buffer

为什么要把这两个放在一起呢？首先看下下面这个图你也许就能够明白了

![](..\NIO\Channels和Buffers.png)

Channel和Buffer在NIO中有很多种类型，下面说一些主要的实现

### a）Channel

- FileChannel
- SocketChannel
- ServerSocketChannel
- DatagramChannel

正如我们常用的，这些通道基本上覆含了文件、网络这两大类的IO

### b）Buffer

- ByteBuffer
- CharBuffer
- DoubleBuffer
- FloatBuffer
- IntBuffer
- LongBuffer
- ShortBuffer

## 2）Selector

Selector的存在就允许单个线程去处理出多Channel，为什么呢？当我们看到下面这个图解就能明白了

![](..\NIO\Channel、Selector与Thread.jpg)

正如我们所看到的，要使用selector去管理channel就必须先把channel注册到selector中去，然后在去调用它的select()方法。select()这个方法一直会阻塞到某个channel有事件就绪，一旦这个方法返回了，那么线程就能去处理这个事件了

# 三、Channel（通道）

为什么不先说Buffer呢？因为我们的Buffer区都是从Channel中读过来的，Channel才是真正的“外交官”。它和传统的流有点一样但是又不太一样，我们可以这样理解，自来水和水管的区别，流只能单向，而Channel是可以双向的。不像流那样直接读写，而是先读到缓存中，再有缓存中写入。

## 1）Channel的实现

- FileChannel：从文件中读写数据
- SocketChannel：通过TCP读写网络中的数据
- ServerSocketChannel：可以监听网络中新额TCP连接，它会对每个新进来的连接建立一个SocketChannel
- DatagramChannel：通过UDP读写网络中的数据（不要问为啥UDP没有监听，因为它是一个无连接的网络通信协议）

## 2）示例

因为Channel和Buffer是要一起使用的，所以实例中可能会有Buffer

```java
@Test
    public void Channel() throws Exception {
        File file = new File("");
        FileInputStream inputStream = new FileInputStream("123.png");
        FileChannel channel = inputStream.getChannel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);//jvm中创建缓冲区
//        ByteBuffer.allocateDirect(1024); 从内存中创建
        int read = channel.read(buffer);
        while (read != -1){
            System.out.println("读到了：" + read + "长度");
            /**
             * 源码：
             * limit = position;
             * position = 0;
             * mark = -1;
             */
            buffer.flip();//翻转模式，之前buffer是读模式，现在要切换到写模式
            while (buffer.hasRemaining()){
                System.out.println(buffer.get());
            }
            /**
             * 源码：
             * position = 0;
             * limit = capacity;
             * mark = -1;
             */
            buffer.clear();
            read = channel.read(buffer);
        }
    }
```

*注意：这里调用get()方法之前调用了flip()方法，这个是必须的。具体说明请看buffer章节*

# 四、Buffer（缓存）

当文件被读到Channel后，想要写入到另外一个地方，但是这不能直接写入的，这里就需要Buffer。缓冲器本质上是一块可以被写入数据，也可以被读取数据的内存；这块内存被NIO包装后就成为了Buffer对象。

## 1）用法

正如我们的上一个实例，

​	1.通过Channel的read()方法把数据写到Buffer中(对channel来说是读，对buffer来说是写)。

​	2.调用Buffer的flip()方法

​	3.从Buffer中获取数据

​	4.清空缓冲区

​	当我们把数据写到buffer中去，buffer会记录下我们写了多少数据进去，这时候buffer是读模式，我们要获取buffer中的数据就需要切换的读模式，这里就需要调动flip()方法进行模式的切换。

​	一旦读完所有的数据，我们就需要清空缓存中的数据，这样是为了让它可以再次被写入；这里有两个方法：

​		1.clear()，清除所有的数据(limit = capacity, position = 0)

​		2.compact()， 清除已经读的数据(这里会计算capacity和limit的差 然后赋值给position，建议去看下源码好理解一点)

## 2）capacity、limit和position

在用法上面说到了几个方法，他们都去改变了limit、position的值。这里就拿这几个属性来说下Buffer的原理。

​	![](..\NIO\Buffer图解.png)

其实buffer的底层就是数组去实现的，它额外映入了mark，position、limit、capacity概念。

### a）capacity

​	它所记录的就是我们一开始开辟内存的大小，这个是固定的，一旦被写满后就需要清除，如果没有清除的话再写入数据就睡报错

### b）position

​	position的初始值是0，当我们写入数据的时候position会记录我们把数据写到哪个位置了，当buffer从写模式切换到读模式后flip()方法，position值会立马被设置为0。

### c）limit

​	在写模式下，limit就等于capacity的值，这表示我们能写多少个数据到buffer中去；当切换到读模式下，limit就是写模式下的position的值，这表示我们最多能读出来多少数据

## 3）Buffer类型

- ByteBuffer
- MappedByteBuffer
- CharBuffer
- DoubleBuffer
- FloatBuffer
- IntBuffer
- LongBuffer
- ShortBuffer

## 4）示例

Buffer的方法有很多，我们通过代码来看一下

```java
@Test
    public void testBuffer() throws Exception{
        FileChannel inChannel = FileChannel.open(Paths.get("123.png"), StandardOpenOption.READ);
        FileChannel outChannel = FileChannel.open(Paths.get("124.png"), StandardOpenOption.READ, StandardOpenOption.WRITE, StandardOpenOption.CREATE);
        //分配缓冲区大小
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        //向buffer中写数据
        int read = inChannel.read(byteBuffer);
        while (read != -1){
            //反转模式 写模式到读模式
            byteBuffer.flip();
            //向通道里面写数据
            outChannel.write(byteBuffer);
            //清除缓冲区
            byteBuffer.clear();
            read = inChannel.read(byteBuffer);
        }
        inChannel.close();
        outChannel.close();
    }
```

# 五、Scatter/Gather（分散和聚集）

Scatter和Gather分别用于描述充Channel中读取和写入。

## 1）Scatter

Scatter是分散，是从Channel中读取写入到不同的Buffer中。

![](..\NIO\Scatter.png)

代码实例

```java
		FileChannel inChannel = FileChannel.open(Paths.get("123.png"), StandardOpenOption.READ);
        ByteBuffer allocate1 = ByteBuffer.allocate((int) inChannel.size());
        ByteBuffer allocate2 = ByteBuffer.allocate((int) inChannel.size());
        ByteBuffer[] arr = {allocate1, allocate2};
        inChannel.read(arr);
```

注：首先将所有的Buffer放到一个数组中，然后将数组做为参数传入到read方法中，read方法会按数组的顺序一个一个的将数据写入到Buffer中

## 2）Gather

Gather是将多个个Buffer中的数据写入到一个Channel中

![](..\NIO\Gather.png)

代码实例

```java
		FileChannel outChannel = FileChannel.open(Paths.get("124.png"), StandardOpenOption.READ, StandardOpenOption.WRITE, StandardOpenOption.CREATE);
        ByteBuffer allocate1 = ByteBuffer.allocate((int) inChannel.size());
        ByteBuffer allocate2 = ByteBuffer.allocate((int) inChannel.size());
        outChannel.write(arr);
```

arr作为参数传入到write方法中，这里写入也会按照数组的顺序依次写入，但是这里写入只会读取Buffer position到limit中的数据

# 六、Channel之间的数据传输

当我们创建好Channel之后，我们就可以在两个Channel之间进行数据的传输，但是必须又有个Channel是FileChannel

它主要通过两个方法去实现

## 1）transferFrom

```java
/**
*src 元数据来自哪里
*position 从哪个位置开始写入
*count 写入多少
*/
public abstract long transferFrom(ReadableByteChannel src,
                                      long position, long count)
        throws IOException;
```

## 2）transferTo

```java
public abstract long transferTo(long position, long count,
                                WritableByteChannel target)
    throws IOException;
```

## 3）示例

```java
FileChannel inChannel = FileChannel.open(Paths.get("123.png"), StandardOpenOption.READ);
FileChannel outChannel = FileChannel.open(Paths.get("124.png"), StandardOpenOption.READ, StandardOpenOption.WRITE, StandardOpenOption.CREATE);
inChannel.transferTo(0, inChannel.size(), outChannel);
outChannel.transferFrom(inChannel, 0, inChannel.size());
```

# 七、Selector

​	selector为我们提供了一个线程监控多个Channel，并能够知道某个通道具体干嘛，这样就让我们能够监听多个网络连接

## 1）为什么需要selector

​	首先我们的内存是有限的，多一条线程可能没什么太大的影响，但是线程越来越多的话，那么内存势必是不够用的。当用一个线程就可以管理所有Channel的时，那内存开销是不是就减少了很多呢？而且还省去了多个线程在CPU中的切换。

## 2）如何创建Selector

```java
Selector open = Selector.open();
```

## 3）如何向Selector中注册Channel

之前说过，要像selector管理Channel就必须让Channel注册到selector中去。

```java
		Selector selector = Selector.open();
        SocketChannel socketChannel = SocketChannel.open();
        //设置为非阻塞模式
        socketChannel.configureBlocking(false);
        //完成注册
        //这里注册需要设置好监听的对象
        //OP_CONNECT 连接
        //OP_ACCEPT 同意
        //OP_READ 读
        //OP_WRITE 写
        socketChannel.register(selector, SelectionKey.OP_READ, SelectionKey.OP_ACCEPT);
```

注：这里不能使用FileChannel， 应为FileChannel不能被设置为非阻塞模式

## 4）示例

```java
public void SelectorTest() throws Exception{
        Selector selector = Selector.open();
        SocketChannel socketChannel = SocketChannel.open();
        //设置为非阻塞模式
        socketChannel.configureBlocking(false);
        //完成注册
        //这里注册需要设置好监听的对象
        //OP_CONNECT 连接
        //OP_ACCEPT 同意
        //OP_READ 读
        //OP_WRITE 写
        socketChannel.register(selector, SelectionKey.OP_READ, SelectionKey.OP_ACCEPT);
        while(true) {
            int readyChannels = selector.selectNow();
            if (readyChannels == 0) continue;
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                if (key.isAcceptable()) {
                    // a connection was accepted by a ServerSocketChannel.
                } else if (key.isConnectable()) {
                    // a connection was established with a remote server.
                } else if (key.isReadable()) {
                    // a channel is ready for reading
                } else if (key.isWritable()) {
                    // a channel is ready for writing
                }
                keyIterator.remove();
            }
        }
    }
```

