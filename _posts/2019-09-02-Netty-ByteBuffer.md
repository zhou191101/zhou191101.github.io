---
layout:     post
title:      "Netty ByteBuffer"
subtitle:   "深入理解Netty ByteBuffer"
date:       2019-09-02
author:     "zhoup"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 大数据
    - Java
    - Netty
---

> This document is not completed and will be updated anytime.

Netty是基于JAVA NIO的rpc框架，因此，探讨Netty的bytebuf之前，不妨先看看Java的nio中的buffer的实现。

## JAVA NIO中的buffer

buffer是一种特定类型的容器。是一个线性的、有限长度的一个特定类型的元素序列。

buffer本身就是一种内存，底层实现上，它实际上是个数组，数据的读和写都是通过buffer来实现的。

除了数组之外，buffer海提供了对于数据的结构化访问方式，并且可以追踪到系统的读写过程。

java中的7种原生数据类型都有各自对应的buffer类型，没有BooleanBuffer类型。

除了它本身的内容，buffer的基本属性是position、limit、capacity。

初始化buffer时，position为0，limit等于capacity等于所分配的buffer大小。
![netty_buffer.png](https://github.com/zhou191101/zhou191101.github.io/blob/master/img/in-post/netty/netty_buffer.png?raw=true)

nio中三个状态属性的含义：

- position 一个buffer的position是指下一个需要被读或者写的元素的索引。一个buffer的position是不为负并且大小不超过limit
- limit 一个buffer中的limit属性指的是第一个不应该被读或者写的元素的索引，每个buffer的limit是不为负的并且大小不超过capacity。
- Capacity buffer所包含元素的个数，每个buffer的capacity是不为负和不可改变的，capacity的大小一般在分配buffer时通过方法ByteBuffer.allocate(512)或者ByteBuffer.allocateDirect(512)创建，其中传人的参数512就是capacity的大小。

相对操作读或者写操作从position 的索引位置开始，position增加的大小就是读或者写操作的元素个数。如果一个请求传递的数据超过了limit限制，那么相对于get操作会抛出BufferUnderflowException异常，相对于put操作会抛出BufferOverflowException异常。当执行相对操作时，所对应的索引会发生改变，即position会随着读或者写操作的进行而增加，读或者写操作所到的位置的索引就位position的值。

绝对操作直接获取明确的索引，不会影响position的值。如果索引超出了限制，那么绝对操作的put和get方法将会抛出IndexOutOfBoundsException异常。当执行绝对读或者写操作时，buffer的position，limit和capacity都不会发生改变。

Buffer.rewind()方法会将position置为0，而limit不变，这样就方便我们重复读取buffer中的数据，当将数据从一个地方传输到另一个地方时，经常将此方法与 compact 方法一起使用。

一旦我们从buffer中读完数据，那么就应该为下一次读取数据到buffer做准备。Buffer为此提供了两个可选的方法：clear() compact。两者的区别是：当调用Buffer.clear()方法时会将position置为0，limit置为capacity，也就是初始化整个buffer，实际上buffer中的数据并没有被清空。如果buffer中还有一些数据未读取完，调用clear()就会将这部分数据丢失掉。因此提供了另外一个方法Buffer.compact()，针对数据未读取完的情况，调用此方法，会对未处理的数据保留。

如下代码片段，首先定义了一个IntBuffer，分配的底层数组大小为10，随机向buffer中添加数据，然后调用

buffer.flip()，（当调用flip()方法后，limit的值将会等于position的值）将buffer中的数据读取并打印出来。

    import java.nio.IntBuffer;
    import java.security.SecureRandom;
    
    public class NioTest1 {
        public static void main(String[] args) {
            IntBuffer buffer = IntBuffer.allocate(10);
            for(int i=0;i<buffer.capacity();i++){
                System.out.println(i+"===");
                int randomNumber = new SecureRandom().nextInt(20);
                buffer.put(randomNumber);
            }
            buffer.flip();
            while (buffer.hasRemaining()){
                System.out.println(buffer.get());
            }
        }
    }

## Netty 中的Bytebuf

Netty提供了对底层buffer容器的抽象类ByteBuf，Netty推荐使用Unpooled中的帮助方法来创建一个底层的buf。netty提供了两张类型的buf，一种是随机访问索引，一种是顺序访问索引。

### 随机访问索引

就像是一个普通的原生byte数组一样，Netty的ByteBuf也是提供了从0开始的索引，和java nio相似，索引的最后一个值就是capacity的值。如果想要迭代的取出buf中的所有元素，你可以使用像以下代码段的方式：

    ByteBuf buffer = ...;
     for (int i = 0; i < buffer.capacity(); i ++) {
         byte b = buffer.getByte(i);
         System.out.println((char) b);
     }

在Netty所提供的方法中，一般以getxxx或者setxxx命名的方法，都是绝对方法，其操作不会改变底层索引的值，相反的，使用readxxx和writexxx命名的方法，都是相对方法，其操作会改变底层索引的值。

### 顺序访问索引

和JAVA Nio中的Buffer不同的是，Netty提供了两个指针变量来--readerIndex和writerInde来区别读和写操作，readerIndex的大小随着read操作的改变而改变，writerIndex的大小随着write操作的改变而改变。以下图示标识了两个不同的索引是如何工作的。

          +-------------------+------------------+------------------+
          | discardable bytes |  readable bytes  |  writable bytes  |
          |                   |     (CONTENT)    |                  |
          +-------------------+------------------+------------------+
          |                   |                  |                  |
          0      <=      readerIndex   <=   writerIndex    <=    capacity

- discardable bytes  指已经被相对操作的读操作访问过的数据，如果需要将可丢弃的数据丢弃，大小为可readerIndex大小。调用discardReadBytes()方法。此方法会将discardable bytes清除（实际上并没有对数据做清除操作，只是将readerIndex置为0）,然后将writerIndex索引左移，如下所示：

     BEFORE discardReadBytes()
     
          +-------------------+------------------+------------------+
          | discardable bytes |  readable bytes  |  writable bytes  |
          +-------------------+------------------+------------------+
          |                   |                  |                  |
          0      <=      readerIndex   <=   writerIndex    <=    capacity


      AFTER discardReadBytes()
    
          +------------------+--------------------------------------+
          |  readable bytes  |    writable bytes (got more space)   |
          +------------------+--------------------------------------+
          |                  |                                      |
     readerIndex (0) <= writerIndex (decreased)        <=        capacity

- readable bytes  指该buffer中可读取的字节大小。一般是相对读操作所未访问到的数据。大小为writerIndex-readerIndex大小。如果没有足够的内容可读取时调用读操作，会产生IndexOutOfBoundsException异常。读操作推荐使用以下方式：
```
     // Iterates the readable bytes of a buffer.
     ByteBuf buffer = ...;
     while (buffer.isReadable()) {
         System.out.println(buffer.readByte());
     }
```

- writable bytes 只该buffer中可分配的字节大小，即buffer中空闲的空间。大小为capacity-writerIndex。如果执行写操作时没有足够的空间，会产生IndexOutOfBoundsException异常。写操作推荐使用以下方式：

```
    // Fills the writable bytes of a buffer with random integers.
     ByteBuf buffer = ...;
     while (buffer.maxWritableBytes() >= 4) {
         buffer.writeInt(random.nextInt());
     }
```

**注意**：通过索引来访问Byte时并不会改变真实的读索引与写索引；我们可以通过ByteBuf的`readerIndex（）`与`writerIndex（）`方法分别直接修改读索引和写索引。

Netty提供了对buffer的清除操作，除了使用上述提到的discardReadBytes()方法之外，还可以使用clear()方法，与之不同的是，clear()方法会将读和写的索引都置为0。他不是真正的清除掉了buffer中的数据，而是清楚了读写指针。操作过程如下所示：

      BEFORE clear()
    
          +-------------------+------------------+------------------+
          | discardable bytes |  readable bytes  |  writable bytes  |
          +-------------------+------------------+------------------+
          |                   |                  |                  |
          0      <=      readerIndex   <=   writerIndex    <=    capacity


​    
​      AFTER clear()
​    
          +---------------------------------------------------------+
          |             writable bytes (got more space)             |
          +---------------------------------------------------------+
          |                                                         |
          0 = readerIndex = writerIndex            <=            capacity

## 派生buffer

你可以使用以下方法对已存在的buffer创建一个视图：

- duplicate()
- slice()
- slice(int, int)
- readSlice(int)
- retainedDuplicate()
- retainedSlice()
- retainedSlice(int, int)
- readRetainedSlice(int)

以上方法都会返回一个新的buffer实例。派生buffer具有自己的读写索引和标记索引。其内部的存储和JDK的ByteBuffer一样也是共享的。这是的派生缓冲区的创建成本是跟低廉的，但这也意味着，如果你修改了它的内容，也同时修改了其对应的源实例。

## 缓冲区类别

Netty ByteBuf所提供的3种缓冲区类别：

1. heap buffer
2. direct buffer
3. composite buffer

### Heap Buffer(堆缓冲区)

这是最常用的类型，ByteBuf将数据存储到jvm的堆空间中，并且将实际的数据放在Byte array中来实现

优点：由于数据是存储在JVM的堆中，因此可以快速的创建与快速的释放，并且它提供了直接访问内部字节数组的方法。

缺点：每次读写数据时，都需要先将数据复制到直接缓冲区中再进行网络传输。

### Direct Buffer（直接缓冲区）

在堆之外直接分配内存空间，直接缓冲区并不会占用堆堆容量空间，因为它是由操作系统在本地内存进行的数据分配。

优点：在使用socket进行数据传递时，性能非常好，因为数据直接位于操作系统的本地内存中，所以不需要从JVM 将数据复制到直接缓冲区，性能很好。

缺点：因为Direct Buffer是直接在操作系统内存中的，所以内存空间的分配与释放要比堆空间更加复杂，而且速度要慢一些。

Netty通过提供内存池来解决这个问题。直接缓冲区并不支持直接通过数组的方式来访问数据。

重点：对于后端的业务消息的编解码来说，推荐使用HeapByteBuf；对于I/O通信线程在读写缓冲区时，推荐使用DirectByteBuf。

### Composite Buffer(复合缓冲区)

缓冲区中可同时具有Heap buffer和direct buffer

## JDK的ByteBuffer与Netty的ByteBuf之间的差异对比：

1. Netty的ByteBuf采用了读写索引分离的策略（readerIndex与writerIndex），一个初始化（里面尚未有任何数据）的ByteBuf的readerIndex与writerIndex值都为0。
2. 当读索引与写索引处于同一个位置时，如果我们继续读取，那么就会抛出IndexOutOfBoundsException。
3. 对于ByteBuf的任何读写操作都会分别单独维护读索引与写索引。maxCapacity最大容量默认的限制就是Integer.MAX_VALUE。

### JDK的ByteBuffer的缺点：

1. final byte[] hb；这是JDk的ByteBuffer对象中用于存储数据的对象声明；可以看到，其字节数组是被声明为final的，也就是长度时固定不变的，一旦分配好后不能动态扩容与收缩；而且当待存储的数据字节很大时就很有可能出现IndexOutOfBoundsException。如果要预防这个异常，那就需要在存储之前完全确定好待存储的字节大小。如果ByteBuffer的空间不足，我们只有一种解决方案：创建一个全新的ByteBuffer对象，然后再将之前的ByteBuffer中的数据复制过去，这一切操作都需要由开发者自己来手动完成。
2. ByteBuffer只使用一个position指针来表示位置信息，在进行读写切换时就需要调用flip方法或是rewind方法，使用起来很不方便。

### Netty的ByteBuf的优点：

1. 存储字节的数组是动态的，其最大值默认是Integer.MAX_VALUE。这里的动态性是体现在write方法中的，write方法在执行时会判断buffer的容量，如果不足则自动扩容。
2. ByteBuf的读写索引时完全分开的，使用起来就很方便。
