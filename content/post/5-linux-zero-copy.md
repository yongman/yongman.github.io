---
title: Linux中Zero-Copy的实现
categories: [技术记录]
tags: ["linux","kernel"]
date: 2021-10-08 01:00:00
---

### 1. 读写系统调用

理解什么是zero-copy，首先从简单的例子中，更容易理解。

```c
read(file, tmp_buf, len);
write(socket, tmp_buf, len);
```

上面的例子，实现从文件中读取数据，然后将读取的数据写入的网络套接字或者文件中。

![](https://cdn.jsdelivr.net/gh/yongman/i@img/picgo/20211008172808.png)

通过上图中，可以看到，read和write函数会触发对应的读写系统调用，系统调用会执行上下文切换，由用户态切换到内核态，系统调用执行完成后回到用户态继续执行。

在read和write系统调用中，数据总共发生了4次拷贝。

- DMA将数据从磁盘读取到内核缓冲区
- CPU将数据从内核缓冲区复制到用户缓冲区
- CPU将数据从用户缓冲区复制到内核缓冲区
- DMA将数据从内核缓冲区复制到硬件设备

### 2. mmap系统调用

mmap系统调用将文件映射到内存空间，映射完成后，可以按内存访问模式来访问文件。

```c
tmp_buf = mmap(file, len);
write(socket, tmp_buf, len);
```

![](https://cdn.jsdelivr.net/gh/yongman/i@img/picgo/20211008173913.png)

整个过程中发生了3次拷贝。

- 执行mmap时，DMA会将文件中的内容按需复制到内核缓冲区
- 由于内核态和用户态mmap是共享的，所以无需将数据再拷贝到用户态
- write系统调用，CPU将数据从内核态直接拷贝到内核态的套接字缓冲区
- DMA将数据从套接字内核缓冲区复制到硬件设备

对比普通的read和write系统调用，这种方式可以避免数据从内核态到用户态的多次拷贝，mmap+write这种方式需要额外处理文件的并发读写，需要自己进行加锁处理来保证不会在读取的过程中出现别的进程对文件进行truncate操作导致异常。

### 3. sendfile系统调用

从linux kernel 2.1开始，内核实现了sendfile系统调用，简化了数据的发送过程，减少数据的拷贝减少上下文切换。

```c
sendfile(socket, file, len);
```

![](https://cdn.jsdelivr.net/gh/yongman/i@img/picgo/20211008181803.png)

- sendfile系统调用触发DMA将数据复制到内核缓冲区
- CPU将数据从内核态复制到套接字缓冲区
- DMA将数据复制到硬件设备

完成数据的发送，整个过程中不需要用户进程执行CPU拷贝数据，减少进程上下文切换。

其实这并不是真正零拷贝技术，因为其实数据从内核态还是依赖CPU进行了数据拷贝，真正的零拷贝是在支持SG-DMA（The scatter-Gather Direct Memory Access）技术，可以进一步减少通过CPU把内核缓冲区里的数据拷贝到socket缓冲区的过程。

从Linux 2.4版本开始，对于支持SG-DMA技术的情况下，sendfile()系统调用的过程如下：

![](https://cdn.jsdelivr.net/gh/yongman/i@img/picgo/20220105230009.png)

整个过程完全不需要CPU参与数据拷贝，两次都由DMA直接搬运。

**参考**

1. https://www.linuxjournal.com/article/6345
2. 图解系统-小林coding
