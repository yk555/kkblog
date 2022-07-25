---
title: JavaNIO
date: 2022-01-06 03:00:00
tags: 
 - Java
 - JavaNIo
 - netty
 - 零拷贝
categories: netty
---



一切还要从JavaNIO说起。

JavaNIO是基于IO多路复用技术，NIO（new IO） 弥补了老式的OIO同步阻塞的不足

## 他们有何不同呢？

- OIO面向流， NIO面向缓冲区

因为是面向流的，所以导致OIO只能顺序读取

- OIO阻塞， NIO非阻塞
- OIO没用选择器，NIO有选择器，但需要底层支持

## JavaNIO有三个核心组件

- Channel 通道
- Buffer 缓冲区
- Selector选择器

可以理解为，**buffer**是存储数据的地方，**channel**是运输数据的载体，**select**用于检查**多个**channel状态变更

### Buffer类

能够覆盖所有基本类型，还包括用于内存映射的MappedByteBuffer

Byte，Char, Double, Float, Int , Long, Short

**buffer类的基本属性**

capacity--容量

position--当前位置

limit--读写的最大上限

mark--标记当前位置，并且能够在reset（）将position回到标记的位置

**重要方法**

allocate（）--创建缓冲区

put（）--写入到缓冲区

flip() -- 翻转，将写模式切换成读模式

get()--从缓冲区取

rewind（）倒带相当于重新读缓冲区里的东西

mark()和reset()

clear（）清空缓冲区并且变为写模式。

### Channel类

四种分类：

1. FileChannel
2. SocketChannel
3. ServerSocketChannel
4. DatagramChannel

### Selector类

channel将自己的事件注册到Selector中

IO事件类型有以下四种：

1. 可读 SelectionKey.OP_READ 
2. 可写 SelectionKey.OP_WRITE
3. 连接 SelectionKey.OP_CONNECT	
4. 接收 SelectionKey.OP_ACCEPT

为什么FileChannel不能被选择器复用？

1. 因为没有继承SelectableChannel

SelectableChannel 提供了实现通道的可选择性所需要的公共方法

1. FilChannel是阻塞的，不能设置成非阻塞

```java
public class NoBlockServer {
	
	public static void main(String[] args) throws IOException {
		
		// 1.获取通道
		ServerSocketChannel server = ServerSocketChannel.open();
		
		// 2.切换成非阻塞模式
		server.configureBlocking(false);
		
		// 3. 绑定连接
		server.bind(new InetSocketAddress(6666));
		
		// 4. 获取选择器
		Selector selector = Selector.open();
		
		// 4.1将通道注册到选择器上，指定接收“监听通道”事件
		server.register(selector, SelectionKey.OP_ACCEPT);
		
		// 5. 轮训地获取选择器上已“就绪”的事件--->只要select()>0，说明已就绪
		while (selector.select() > 0) {
			// 6. 获取当前选择器所有注册的“选择键”(已就绪的监听事件)
			Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
			
			// 7. 获取已“就绪”的事件，(不同的事件做不同的事)
			while (iterator.hasNext()) {
				
				SelectionKey selectionKey = iterator.next();
				
				// 接收事件就绪
				if (selectionKey.isAcceptable()) {
					
					// 8. 获取客户端的链接
					SocketChannel client = server.accept();
					
					// 8.1 切换成非阻塞状态
					client.configureBlocking(false);
					
					// 8.2 注册到选择器上-->拿到客户端的连接为了读取通道的数据(监听读就绪事件)
					client.register(selector, SelectionKey.OP_READ);
					
				} else if (selectionKey.isReadable()) { // 读事件就绪
					
					// 9. 获取当前选择器读就绪状态的通道
					SocketChannel client = (SocketChannel) selectionKey.channel();
					
					// 9.1读取数据
					ByteBuffer buffer = ByteBuffer.allocate(1024);
					
					// 9.2得到文件通道，将客户端传递过来的图片写到本地项目下(写模式、没有则创建)
					FileChannel outChannel = FileChannel.open(Paths.get("2.png"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
					
					while (client.read(buffer) > 0) {
						// 在读之前都要切换成读模式
						buffer.flip();
						
						outChannel.write(buffer);
						
						// 读完切换成写模式，能让管道继续读取文件的数据
						buffer.clear();
					}
				}
				// 10. 取消选择键(已经处理过的事件，就应该取消掉了)
				iterator.remove();
			}
		}
		
	}
}
```



```java
public class NoBlockClient {

    public static void main(String[] args) throws IOException {

        // 1. 获取通道
        SocketChannel socketChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 6666));

        // 1.1切换成非阻塞模式
        socketChannel.configureBlocking(false);

        // 1.2获取选择器
        Selector selector = Selector.open();

        // 1.3将通道注册到选择器中，获取服务端返回的数据
        socketChannel.register(selector, SelectionKey.OP_READ);

        // 2. 发送一张图片给服务端吧
        FileChannel fileChannel = FileChannel.open(Paths.get("X:\\Users\\ozc\\Desktop\\面试造火箭\\1.png"), StandardOpenOption.READ);

        // 3.要使用NIO，有了Channel，就必然要有Buffer，Buffer是与数据打交道的呢
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        // 4.读取本地文件(图片)，发送到服务器
        while (fileChannel.read(buffer) != -1) {

            // 在读之前都要切换成读模式
            buffer.flip();

            socketChannel.write(buffer);

            // 读完切换成写模式，能让管道继续读取文件的数据
            buffer.clear();
        }


        // 5. 轮训地获取选择器上已“就绪”的事件--->只要select()>0，说明已就绪
        while (selector.select() > 0) {
            // 6. 获取当前选择器所有注册的“选择键”(已就绪的监听事件)
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();

            // 7. 获取已“就绪”的事件，(不同的事件做不同的事)
            while (iterator.hasNext()) {

                SelectionKey selectionKey = iterator.next();

                // 8. 读事件就绪
                if (selectionKey.isReadable()) {

                    // 8.1得到对应的通道
                    SocketChannel channel = (SocketChannel) selectionKey.channel();

                    ByteBuffer responseBuffer = ByteBuffer.allocate(1024);

                    // 9. 知道服务端要返回响应的数据给客户端，客户端在这里接收
                    int readBytes = channel.read(responseBuffer);

                    if (readBytes > 0) {
                        // 切换读模式
                        responseBuffer.flip();
                        System.out.println(new String(responseBuffer.array(), 0, readBytes));
                    }
                }

                // 10. 取消选择键(已经处理过的事件，就应该取消掉了)
                iterator.remove();
            }
        }
    }

}
```

## Linux下的IO多路复用技术

文件句柄数的限制

默认1024,也就是一个进程最多接受1024个socket连接

从基础开始讲起，文件句柄，也叫文件描述符。

在linux中文件分为，普通文件、目录文件、链接文件、和设备文件

文件描述符是内核为了高效管理已被打开的文件所创建的索引。是一个非负整数（通常是小数）

ulimit 用来显示-修改当前用户进程的一些信息，-n是与文件句柄相关。

三种修改句柄方式

ulimit -n 10000 当终端工具退出时就会失效

向 /etc/rc.local添加 ulimit -SHn 10000

终极 解除 修改极限配置文件

soft nofile 10000

hard nofile 10000 ElasticSearch必须修改

此外，linux对文件的操作实际上是通过**文件描述符**，而**IO复用模型**就是通过一个线程监控**多个**文件描述符，当某个文件描述符准备就绪时，就去通知程序做处理。这样不仅单个连接处理的快，还能处理更多的连接。



linux下的IO复用模型就是select/epoll函数

## select和epoll的区别

**select**

函数支持的最大的连接数是1024或者2048，因为在select函数要传入fd_set参数（看操作系统的位数）

fd_set是bitmap的数据结构，可以简单理解为只要位为0，就是数据没到缓冲区，为1 到了缓冲区

select函数做的就是每次将fd_set遍历，有变化就通知处理。

**epoll**

使用epoll_event结构体来处理，不存在最大连接数的限制。并且不是通过遍历的方式，简单理解就是epoll把就绪的文件描述符专门维护了一块空间，每次从就绪列表里边拿。

## 零拷贝

和JavaNIO相关的另一个概念是**零拷贝**

传统IO情况下，当用户程序发起一次读请求，会调用read相关的系统函数，然后会从用户态切换到内核态，随后CPU告诉DMA去磁盘把数据拷贝到内核空间。内核缓冲区有数据后，CPU就把内核缓冲区数据拷贝到用户缓冲区，最终用户程序拿到数据。

DMA是直接内存访问，允许外部设备直接与内存设备进行数据传输不需要CPU参与的技术

为了内核安全，所以将操作系统划分为**用户空间**和**内核空间,**读数据会有状态切换。

**零拷贝**是指计算机执行IO操作时，CPU不需要将数据从一个存储区域复制到另一个存储区域，从而可以减少上下文切换以及CPU的拷贝时间。它是一种I/O操作优化技术。

- mmap--内核缓冲区与用户缓冲区共享
- sendfile--系统底层函数的支持



下面是比较详细的解释

### 传统IO过程

- read ：把数据从磁盘读取到内核缓冲区，再拷贝到用户缓冲区
- write：先把数据写入到socket缓冲区，最后写入网卡设备

![img](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/1657705520995-6609b87a-c626-4bf0-8a58-2e4c36e4f385.png)

过程

1. 用户程序通过read函数向内核发起IO，从用户态切换为内核态，然后再通过DMA将数据从磁盘读到内核缓冲区
2. CPU将内核缓冲区数据拷贝到用户缓冲区，然后read函数返回，从内核态切换到用户态。
3. 用户程序通过write函数向内核发起IO，从用户态切换到内核态，然后CPU将数据从用户缓冲区拷贝到内核socket缓冲区，然后write返回，从内核态切换到用户态。
4. 最后异步传输到网卡。

可以看出，发生了四次状态切换，CPU拷贝两次。

https://blog.csdn.net/a745233700/article/details/122660332





**零拷贝**指在进行数据IO时， 数据在**用户态**下经历了**零次CPU拷贝**。通过减少内核缓冲区和用户缓冲区之间不必要的cpu数据传输，与用户态和内核态的切换磁环，降低开销提高性能。零拷贝基于PageCache，提升访问缓存数据的性能，同时IO合并与预读（顺序读比随机读性能好）解决机械磁盘寻址慢。



Linux中的零拷贝方式



### mmap 

mmap就是操作系统把内核缓冲区与用户程序共享，也可以说是将用户空间内存映射到内核空间，映射成功后，用户对这段内存区域的修改可以直接反映到内核空间，反过来也一样。正因如此，就不需要在用户态与内核态之间拷贝数据。

![img](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/1657758444170-31a5ae57-0114-4af0-810c-54b1a8980c8e.png)

过程：

1. 用户程序调用mmap发起IO，从用户态切换到内核态，通过DMA将数据从磁盘拷贝到内核缓冲区。
2. mmap返回，从内核态切换到用户态，不需要将数据从内核缓冲区复制到用户缓冲区，因为共享。
3. 用户程序通过write发起IO，从用户态切换到内核态，CPU将数据从内核缓冲区复制到内核socket缓冲区，返回内核态切换为用户态。
4. DMA异步将socket缓冲区拷贝到网卡



### sendfile

由于调用read或者write一定会发生两次上下文切换，所以想要减少状态切换，那就把read和write合并起来，在内核中完成磁盘与网卡的数据交互

![img](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/1657759161290-afdf51ec-0666-4a0a-84f4-368ee15c953e.png)

过程：

1. 用户程序发起sendfile，用户态切换到内核态，DMA将数据从磁盘复制到内存缓冲区
2. CPU将数据从内核缓冲区复制到socket缓冲区
3. sendfile系统调用返回，从内核态切换到用户态
4. DMA异步将socket数据复制到网卡





### 带DMA收集拷贝功能的sendfile

sendfile升级后，引入了SG-DMA，就是对DMA拷贝加入了scatter、gather操作，可以直接将内存缓冲区数据复制到网卡，无需再复制到socket缓冲区，减少CPU拷贝次数。

![img](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/2054/1657759464898-fe5e078a-e722-4c98-a31f-cddcbe443c4f.png)

过程：

1. 用户程序发起sendfile调用，从用户态切换到内核态，DMA将数据从磁盘拷贝到内核缓冲区
2. 不需要CPU将数据复制到socket缓冲区，而是将文件描述符信息复制到socket缓冲区，该描述符包含 内核缓冲区的内存地址、内核缓冲区的偏移量
3. sendfile返回，从内核态切换到用户态
4. DMA根据socket缓冲区中描述的地址和偏移量直接将内核缓冲区复制到网卡。

零拷贝的缺点：不允许进程对文件内容做加工再发送，比如数据压缩





### 零拷贝的应用场景



**JavaNIO**

1. mmap+ write

上文提到过buffer中有个叫mappedBuffer的家伙，同时Filechannel提供了map方法，可以再一个打开的文件和MappedByteBuffer之间建立一个虚拟内存映射，map底层是通过mmap实现的，因此磁盘文件到缓冲区后，用户和内核共享缓冲区。

```java
public class MmapTest {
 
    public static void main(String[] args) {
        try {
            FileChannel readChannel = FileChannel.open(Paths.get("./jay.txt"), StandardOpenOption.READ);
            MappedByteBuffer data = readChannel.map(FileChannel.MapMode.READ_ONLY, 0, 1024 * 1024 * 40);
            FileChannel writeChannel = FileChannel.open(Paths.get("./siting.txt"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
            //数据传输
            writeChannel.write(data);
            readChannel.close();
            writeChannel.close();
        }catch (Exception e){
            System.out.println(e.getMessage());
        }
    }
}
```



1. sendfile

   FileChannel 的 transferTo、transferFrom 如果操作系统底层支持的话，transferTo、transferFrom也会使用 sendfile 零拷贝技术来实现数据的传输

```java
@Override
public long transferFrom(FileChannel fileChannel, long position, long count) throws IOException {
   return fileChannel.transferTo(position, count, socketChannel);
}

public class SendFileTest {
    public static void main(String[] args) {
        try {
            FileChannel readChannel = FileChannel.open(Paths.get("./jay.txt"), StandardOpenOption.READ);
            long len = readChannel.size();
            long position = readChannel.position();
            
            FileChannel writeChannel = FileChannel.open(Paths.get("./siting.txt"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);
            //数据传输
            readChannel.transferTo(position, len, writeChannel);
            readChannel.close();
            writeChannel.close();
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }
    }
}
```

**Netty框架**

Netty 的零拷贝主要体现在下面五个方面：



（1）在网络通信上，Netty 的接收和发送 ByteBuffer 采用直接内存，使用堆外直接内存进行 Socket 读写，不需要进行字节缓冲区的二次拷贝。如果使用传统的堆内存进行 Socket 读写，JVM 会将堆内存 Buffer 拷贝一份到直接内存中（为什么拷贝？因为 JVM 会发生 GC 垃圾回收，数据的内存地址会发生变化，直接将堆内的内存地址传给内核，内存地址一旦变了就内核读不到数据了），然后才写入 Socket 中。相比于堆外直接内存，消息在发送过程中多了一次缓冲区的内存拷贝。



（2）在文件传输上，Netty 的通过 FileRegion 包装的 FileChannel.tranferTo 实现文件传输，它可以直接将文件缓冲区的数据发送到目标 Channel，避免了传统通过循环 write 方式导致的内存拷贝问题。



（3）在缓存操作上，Netty 提供了CompositeByteBuf 类，它可以将多个 ByteBuf 合并为一个逻辑上的 ByteBuf，避免了各个 ByteBuf 之间的拷贝。



（4）通过 wrap 操作，我们可以将byte[]数组、ByteBuf、ByteBuffer等包装成一个Netty ByteBuf对象，进而避免了拷贝操作。



（5）ByteBuf 支持 slice 操作，因此可以将 ByteBuf 分解为多个共享同一个存储区域的 ByteBuf，避免了内存的拷贝。

[
](https://blog.csdn.net/a745233700/article/details/122660332)



**kafka**

Kafka 的索引文件使用的是 mmap + write 方式，数据文件使用的是 sendfile 方式

[零拷贝1](https://blog.csdn.net/qq_44027353/article/details/121306224)

[零拷贝2](https://blog.csdn.net/a745233700/article/details/122660332)

[零拷贝3](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247520212&idx=1&sn=2d7a19f884fb59e3e961c640c73b8364&chksm=cea1c01ff9d649097dccaef7ba99956250f3bb9998949009b93de0043c023b05259824b9c944&mpshare=1&scene=1&srcid=0420LAxjPloJMzCu9HKGYXMz&sharer_sharetime=1650438077020&sharer_shareid=2e92f088aa52899c6eaf279275db8190&key=066c8b4d62994f4a663099da0975284f295b4b282a9acdd1c75d995ec0c06d7cfe9cd98ed4d05a4512fafd0e0616150a28829e59088dd678d804e2b1f65fdcdce4ae06b6412a0bc40e1e0bf7bb9aec85497c8983188845aea63451b37a63b5cd662409f52fc036c54ded636968ed92c73c7b394b909e3790145b4b4e3035ee94&ascene=1&uin=MTE4OTU2NTIxNQ%3D%3D&devicetype=Windows-QQBrowser&version=6103000b&lang=zh_CN&exportkey=A3zMFqmRKu%2F%2F6LRmGWqUouM%3D&acctmode=0&pass_ticket=Ie8HXClpM2EmWqTabSqVSGwtMO0c3i0ZbY%2FVPSygbfCYdwbNH6PIDhwzvZ6mGEQs&wx_header=0)

