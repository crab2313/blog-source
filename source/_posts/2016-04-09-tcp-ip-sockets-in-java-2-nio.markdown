---
layout: post
title: "TCP/IP Sockets in Java:2 NIO"
date: 2016-04-09T15:23:05+08:00
---

# Buffer
`Buffer`对象代表一个缓冲区，是`Channel`中进行传输的基本单元。NIO中`Channel`与`Selector`的组合为我们提供了更加灵活的非阻塞IO解决方案。Buffer可看作一个定长数组，这个数组有四个索引，分别是position，mark，capacity及limit。

*   capacity是这个Buffer对象的容量，是不可变的。
*   limit的值代表第一个不可读写的数组元素的位置。
*   position是当前的操作位置。读写操作都是以position为起始位置。
*   mark的值是用户自定的一个标记。

Buffer对象内部的不变式：

    mark <= position <= limit <= capacity

注意mark的值可为负，表示其为未定义的。

Buffer对象**不是线程安全的**，在多线程环境下需要同步。Buffer对象本身为一个抽象类，我们一般使用其子类`ByteBuffer`。`Buffer`类定义了上面四个索引的setter和getter，还提供了三个用于帮助我们设置索引的常用方法。

*   clear()使position=0，mark=-1，limit=capacity。这个操作是清空了这个Buffer
*   clip()使limit=position，position=0,mark=-1。这个操作为从Buffer开头读取数据做了准备
*   rewind()仅仅使position=0,mark=-1。使用此操作可以重复读取一个Buffer中的数据

参考源码：


``` java
package java.nio;

public abstract class Buffer {
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;

    
    // used only by direct buffers
    long address;

    // package private
    Buffer(int mark, int pos, int lim, int cap) {
        if (cap < 0)
            throw new IllegalArgumentException("Negative capacity: " + cap);
        this.capacity = cap;
        position(position);
        limit(lim);
        if (mark >= 0) {
            if (mark > pos) 
                throw new IllegalArgumentException("mark > position: ("
                                            + mark + " > " + pos + ")");
            this.mark = mark;
        }
    }

    
    // getters
    public final int capacity() { return capacity; }
    public final int position() { return position; }
    public final int limit() { return limit; }

    // setters
    public final Buffer position(int newPosition) { ... }
    public final Buffer limit(int newLimit) { ... }
    
    // mark current position
    public final Buffer mark() { mark = position; return this; }
    // reset buffer's position to the previously-marked position
    public final Buffer reset() { 
        if (mark < 0)
            throw new InvalidMarkException();
        position = mark;
        return this;
    }

    // three functions for preparing accessing the buffer
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }

    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
    
    public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }

    // returns the number of elements between the 
    // current position and the limit
    public final int remaining() { return limit - position; }
    public final int hasRemaining() { return position > limit; }


    // more abstract methods
    ... 
}
```

我们可以使用`ByteBuffer.allocate()`及`ByteBuffer.wrap()`工厂方法创建`ByteBuffer`对象，其中allocate重新创建一个数组，而wrap则会利用一个原有数组。注意这个数组所占用的内存的对象为JVM中的内存，并不能直接与操作系统系统交互，也就是说进行IO操作的时候还需要将这个Buffer对象的数据复制到另一个操作系统提供的缓冲区中，造成额外开销。我们可以使用`Buffer.allocateDirect(BUFFERSIZE)`这个方法创建一个Direct Buffer，这个Buffer对象就是直接利用的操作系统的缓冲区，避免了额外开销。但是需要注意这个方法并不能保证创建的Buffer一定是Direct Buffer，所以我们需要自己使用`isDirect()`方法来确认该Buffer是否是`Direct Buffer`。

ByteBuffer支持两种类型的`get`与`put`方法，分为相对与绝对。同时`ByteBuffer`也提供了特殊的get与set，可以完成基本类型与byte的转化，默认字节序为大端。

相对方式：

``` java
byte get();
ByteBuffer get(byte[] dst);
ByteBuffer get(byte[] dst, int offset, int length);
ByteBuffer put(byte b);
ByteBuffer put(byte[] src);
ByteBuffer put(byte[] src, int offset, int length);
ByteBuffer put(ByteBuffer src);
```

绝对方式：

``` java
byte get(int index);
ByteBuffer put(int index, byte);
```
# SocketChannel And ServerSocketChannel
`SocketChannel`对应着`Socket`，`ServerSocketChannel`对应着`ServerSocket`。只不过它们操作的对象从一个流转变成了`ByteBuffer`。这两个类都使用`open()`这个工厂方法创建对象实例。`configureBlocking()`方法可以设置是否阻塞。

# Selector
`Selector`对象可以参考POSIX中的select接口，他实现了非阻塞IO的基本框架。我们创建一个Selector，向一个`Channel`注册这个`Selector`，这个`Selector`就负责监视这个`Channel`。我们可以使用一个`Selector`管理大量的`Channel`，同过调用`select()`这个阻塞的方法，我们可以接到在`Channel`可以接受特定行为的通知。

# SelectionKey
`SelectionKey`代表一个`Selector`与`Channel`的联系。我们在注册`Selector`给`Channel`时就会得到一个`SelectionKey`。一个`Channel`可以自己决定它感兴趣的事件，分为`OP_ACCEPT`，`OP_CONNECT`，`OP_READ`，`OP_WRITE`。

``` java
import java.io.*;
import java.net.*;
import java.nio.channels.*;
import java.util.*;

public class TCPServerSelector {
    private static final int BUFSIZE = 256;
    private static final int TIMEOUT = 3000;

    public static void main(String[] args) throws IOException {
        if (args.length < 1) 
            throw new IllegalArgumentException("Parameters: <Port> ...");

        Selector selector = Selector.open();

        for (String arg : args) {
            ServerSocketChannel listenChannel = ServerSocketChannel.open();
            listenChannel.socket()
                .bind(new InetSocketAddress(Integer.parseInt(arg)));
            listenChannel.configureBlocking(false);
            listenChannel.register(selector, SelectionKey.OP_ACCEPT);
        }

        TCPProtocol protocol = new EchoSelectorProtocol(BUFSIZE);
        while (true) {
            if (selector.select(TIMEOUT) == 0) {
                System.out.print(".");
                continue;
            }

            Iterator<SelectionKey> keyIter = 
                selector.selectedKeys().iterator();
            while (keyIter.hasNext()) {
                SelectionKey key = keyIter.next();
                if (key.isAcceptable()) 
                    protocol.handleAccept(key);

                if (key.isReadable())
                    protocol.handleRead(key);

                if (key.isValid() && key.isWritable())
                    protocol.handleWrite(key);

                keyIter.remove();
            }
        }
    }
}

```

``` java
import java.nio.*;
import java.nio.channels.*;
import java.io.*;

public class EchoSelectorProtocol implements TCPProtocol {
    private int bufSize;

    public EchoSelectorProtocol(int bufSize) {
        this.bufSize = bufSize;
    }

    public void handleAccept(SelectionKey key) throws IOException {
        SocketChannel clientChan = 
            ((ServerSocketChannel)key.channel()).accept();

        clientChan.configureBlocking(false);
        clientChan.register(key.selector(), SelectionKey.OP_READ, 
                ByteBuffer.allocate(bufSize));
    }

    public void handleRead(SelectionKey key) throws IOException {
        SocketChannel clientChan = (SocketChannel)key.channel();
        ByteBuffer buf = (ByteBuffer)key.attachment();
        long bytesRead = clientChan.read(buf);
        if (bytesRead == -1) {
            clientChan.close();
        } else if (bytesRead > 0) {
            key.interestOps(SelectionKey.OP_READ | SelectionKey.OP_WRITE);
        }
    }

    public void handleWrite(SelectionKey key) throws IOException {
        ByteBuffer buf = (ByteBuffer)key.attachment();
        buf.flip();
        SocketChannel clientChan = (SocketChannel)key.channel();
        clientChan.write(buf);
        if (!buf.hasRemaining()) {
            key.interestOps(SelectionKey.OP_READ);
        }
        buf.compact();
    }
}

```
