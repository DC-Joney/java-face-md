# Kafka

## Kafka 概览

kafka 需要注意的几个点如下：

- consumer group 的分配策略
- 生产者的ISR机制
- 生产者的ACK机制
- 事务消息 + Exaclty Onece
- 文件存储
- Kafka 生产者分区策略、消息发送拦截器，消息保存Offset
- Kafka 消息发送的 client 端流程
- kafka 消息积压以及问题如何排查



Kafka 主要由以下几个部分组成：

![image-20221004203916860](assets/image-20221004203916860.png)

- 1）Producer ：消息生产者，就是向kafka broker发消息的客户端； 
- 2）Consumer ：消息消费者，向kafka broker取消息的客户端； 
- 3）Consumer Group （CG）：消费者组，由多个consumer组成。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。 
- 4）Broker ：一台kafka服务器就是一个broker。一个集群由多个broker组成。一个broker可以容纳多个topic。
-  5）Topic ：可以理解为一个队列，生产者和消费者面向的都是一个topic； 
- 6）Partition：为了实现扩展性，一个非常大的topic可以分布到多个broker（即服务器）上，一个topic可以分为多个partition，每个partition是一个有序的队列；
-  7）Replica：副本，为保证集群中的某个节点发生故障时，该节点上的partition数据不丢失且kafka仍然能够继续工作，kafka提供了副本机制，一个topic的每个分区都有若干个副本，一个leader和若干个follower
- 8）leader：每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是leader
- 9）follower：每个分区多个副本中的“从”，实时从leader中同步数据，保持和leader数据的同步。leader发生故障时，某个follower会成为新的follower





## Kafka 副本

Kafka 中副本分为：Leader 和 Follower。Kafka 生产者只会把数据发往 Leader， 然后 Follower 找 Leader 进行同步数据

**Kafka 副本作用：提高数据可靠性**。 kafka的topic被分成多个分区，分区是基本的数据块。每个分区可以有多个副本，其中一个是leader。所有事件都是发给leader，或者直接从leader读取数据。其他副本只需要与leader保持同步，并及时复制最新的数据。

Kafka 分区中的所有副本统称为 AR（Assigned Repllicas）。  AR = ISR + OSR 

- ISR，表示和 Leader 保持同步的 Follower 集合。如果 Follower 长时间未向 Leader 发送通信请求或同步数据，则该 Follower 将被踢出 ISR。该时间阈值由 replica.lag.time.max.ms
- OSR**，**表示 Follower 与 Leader 副本同步时，延迟过多的副本。 

> Kafka 默认副本 1 个，生产环境一般配置为 2 个，保证数据可靠性；太多副本会 增加磁盘存储空间，增加网络上数据传输，降低效率。 



**Leader维护了一个动态的in-sync replica set（ISR）**，意为和leader保持同步的follower集合。当ISR中的follower完成数据同步后，leader就会发送ack。如果follower长时间未向leader同步数据，则该follower将被踢出ISR，该时间阈值由[replica.lag.time.max.ms](https://link.segmentfault.com/?enc=SKhXg0gj%2B%2FpyEwpg34EWrQ%3D%3D.OaHDFCay6lHsVkVxCo%2B%2BetZX0o%2FOr5H4Drvh%2FR0ARNU%3D)参数设定。Leader不可用时，将会从ISR中选举新的leader。



每一个partition 都维护了一个HW 与LEO，HW 表示消费者可以看到的最大的Offset，LEO：每个副本最大的offset

![img](assets/1460000039010770.png)

**（1）follower故障**

 follower发生故障后会被临时踢出ISR，待该follower恢复后，follower会读取本地磁盘记录的上次的HW，并将log文件高于HW的部分截取掉，从HW开始向leader进行同步。等该follower的LEO大于等于该Partition的HW，即follower追上leader之后，就可以重新加入ISR了。

![image-20221004212811763](assets/image-20221004212811763.png)



**（2）leader故障**

 leader发生故障后，会从ISR中选出一个新的leader，之后为了保证多个副本之间的数据一致性，其余的follower会先将各自的log文件高于HW的部分截掉，然后从新的leader同步数据。

 注意：这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复

![image-20221004212822602](assets/image-20221004212822602.png)



## Kafka 选举

Kafka的选举分为三种，分别是Controller 选举，Partition 选举以及Consumer leader 选举

### Controller 选举

在启动 Kafka 系统时候, 其中一个 Broker 会被选举为控制器, 负责管理主题分区和副本的状态, 还会执行重分配的任务。

1) 第一个启动的节点，会在 Zookeeper 系统里面创建一个临时节点 /controller ，并写入该节点的注册信息，使该节点成为控制器。

2) 其他的节点在陆续启动时，也会尝试在 Zookeeper 系统中创建 /controller 节点，但是 /controller 节点已经存在，所以会抛出 “创建/controller节点失败异常” 的信息。创建失败的节点会根据返回的结果，判断出在 Kafka 集群中已经有一个控制器被成功创建了，所以放弃创建 /controller 节点，这样就确保了 Kafka 集群控制器的唯一性。

3) 其他的节点，也会在控制器上注册相应的监听器，各个监听器负责监听各自代理节点的状态变化。当监听到节点状态发生变化时，会触发相应的监听函数进行处理。

各个节点公平竞争抢占 Zookeeper 系统中创建 /controller临时节点，最先创建成功的节点会成为控制器，并拥有选举主题分区Leader节点的功能

 Kafka 使用 Zookeeper 来维护集群 Brokers 的信息，每个 Broker 都有一个唯一的标识**`broker.id`**，用于标识自己在集群中的身份。Brokers 会通过 Zookeeper 选举出一个叫**`Controller Broker`**节点，它除了具备其它Brokers的功能外，还**负责管理主题分区及其副本的状态**。





## Kafka Producer

### 发送流程

在消息发送的过程中，涉及到了**两个线程——**main **线程和** Sender **线程**。在 main 线程 中创建了**一个双端队列** RecordAccumulator。main 线程将消息发送给 RecordAccumulator，Sender 线程不断从 RecordAccumulator 中拉取消息发送到 Kafka Broker

Kafka的发送流程如下：

![image-20221004155529974](assets/image-20221004155529974.png)

为了提高效率，消息被分批次写入kafka。批次就是一组消息，这些消息属于同一个主题和分区。（如果每一个消息都单独穿行于网络，会导致大量的网络开销，把消息分成批次传输可以减少网络开销。不过要在时间延迟和吞吐量之间做出权衡：批次越大，单位时间内处理的消息就越多，单个消息的传输时间就越长）。批次数据会被压缩，这样可以提升数据的传输和存储能力，但要做更多的计算处理

- **batch.size：**只有数据积累到batch.size后，sender才会发送数据。（单位：字节，注意：不是消息个数）。
- [linger.ms](https://link.segmentfault.com/?enc=Uths1l6ghqFOboI%2BVvRRoA%3D%3D.bQFVlbfhW3SbqAfCyCzS0HX%2B5j5c1YMbWz7%2FNYBXq5U%3D)**：**如果数据迟迟未达到batch.size，sender等待 [linger.ms](https://link.segmentfault.com/?enc=h3TZ3o0YSYrkbYQGphoAsw%3D%3D.y%2BBR7lAqxYSqrUIV7n0DNsaX5gg3dI%2Bj2fz0lnL3ZPg%3D)之后也会发送数据。（单位：毫秒）。
- [client.id](https://link.segmentfault.com/?enc=rhjNFkGmUug2hwx0WBRbYQ%3D%3D.UpNfbww5Q%2F2I22vu9MQNzlUWrlF2zAno%2FWs2EhMww5Y%3D)**：**该参数可以是任意字符串，服务器会用它来识别消息的来源，还可用用在日志和配额指标里。
- [max.in](https://link.segmentfault.com/?enc=uXQWuSzzxFwWtMsB52V8Xw%3D%3D.HVTyhc3CQiCD4GVVpW7idQ%3D%3D)**.flight.requests.per.connection：**该参数指定了生产者在收到服务器响应之前可以发送多少个消息。它的值越高，就会占用越多的内存，不过也会提升吞吐量。**把它设置为1可以保证消息时按发送的顺序写入服务器的，即使发生了重试。**

![图片](https://mmbiz.qpic.cn/mmbiz_png/FrBePKkiazppn1MKTI37kEBia3ib6zYGoXELelWbxrHBHwX4z3mcX9UyT9yWN9U0SEnOG5H9BxhYtU7qbdMjjqqhQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

1) 首先来一条消息后,生产者源码里面会对消息进行封装成 ProducerRecord对象。 

2) 封装成对象后会对该对象进行序列化[涉及网络传输], 调用Serializer组件进行序列化, 序列化后进行发送。

3) 在发送前要确定一件事, 到底要把这条消息发送到哪个主题的哪个分区, 这个时候就需要通过 Partitioner 分区器 从 Kafka Broker集群中获取集群元数据,  获取到元数据后就可以进行发送了。

4) 在0.8版本之前, 这个时候来了一条消息就会封装成一个请求发送到Broker, 这种情况下, 性能是非常差的, 在0.8版本之后, 进行简单的改进, 性能得到了指数级上升, 即来了一条消息后不会立马发送出去, 而是先写入到一个缓存(RecordAccumulator)队列中,封装成一个个批次(RecordBatch)。

5) 这个时候会有一个sender线程会将多个批次封装成一个请求(Request), 然后进行发送, 这样会减少很多请求,提高吞吐量。**这个时候有个问题, 一条消息过来后没有立即发送出去,而是封装成了批次, 这样会不会有延迟的问题, 默认的batch.size是16K, 写满会立即发送, 如果写不满, 也会在规定的时间进行发送(linger.ms = 500ms)**

6) 发送的时候 每个Request请求对应多路复用器(Selector)中的每个kafka channel 然后将数据发送给Broker集群

7) 在封装Batch批次和Request请求的过程中, 还涉及一个重要的设计理念即内存池方案, 在后面的服务端内存池部分进行详细说明



### 默认分区策略

![image-20221004212417345](assets/image-20221004212417345.png)



### 可靠性保证

- **0：** producer不等待broker的ack，这一操作提供了一个最低的延迟，broker一接收到还没写入磁盘就已经返回，当broker故障时可能丢失数据；
-  **1：** producer等待leader的ack，partition的leader落盘成功后返回ack，如果在follower同步成功之前leader故障，那么将会丢失数据；
-  **-1（all）：**producer等待broker的ack，partition的leader和ISR里的follower全部落盘成功后才返回ack。但是如果在follower同步完成后，broker发送ack之前，leader发生故障，那么会造成重复数据。（极端情况下也有可能丢数据：ISR中只有一个Leader时，相当于1的情况）

![image-20221004213839841](assets/image-20221004213839841.png)



### 内存池

Kafka client 由于在发送消息时为了增加性能减少网络传输的次数，会在客户端缓存需要发送的消息，为了避免频繁的创建内存，减少gc的次数，Kafka提出了内存池的概念，既在发行消息时，kafka会首先从内存池中查找是否有已经创建好的内存块，如果没有则会开辟一块新的内存放入到内存池中

![图片](https://mmbiz.qpic.cn/mmbiz_png/FrBePKkiazppn1MKTI37kEBia3ib6zYGoXEmG0clQsW211cHS33aFMHBC2eNw1YRFwiaf8NugZamu6JhsT1StyVEBw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

1) 这里简化下流程, 来一条消息会先进行封装然后序列化最后会计算出分区号, 并把这个消息存储到缓存里面

2) 这个缓存里面也是有设计的 即批次队列, 那么这个批次队列是使用什么策略存储呢? **一个分区对应一个队列, 这里有个重要的数据结构:Batches, 这个数据结构是Key-value形式, key是消息主题的分区, value是一个队列, 里面存储的发送到对应分区的批次**

3) 那么假设这个时候 我们有个2个topic, 每个topic有2个分区, 那么是不是总共有4个的分区即4个队列, 每个队列里面都有一个个批次, 这个时候消息算出来分区后就会写入队列的最新一个批次

4) Sender线程就会检测这个批次(Batch)是否已经写满,或者时间是否到达, 如果满足Sender线程就会取出封装成Request就会发送

5) 封装批次会用到内存, Sender发送完毕内存会进行回收, 在Java中如果频繁操作内存和回收,会遇到头疼的FullGC的问题, 工作线程的性能就会降低, 整个生产者的性能就会受到影响, **Kafka的解决方案就是内存池, 对内存块的使用跟数据库的连接池一样**

6) 整个Buffer Poll 内存池大小是**32M** , 内存池分为两个部分, 一个部分是内存队列, 队列里面有一个个**内存块(16K)**, 另外一部分是可用内存,  一条消息过来后会向内存池申请内存块, 申请完后封装批次并写入数据, sender线程就会发送并响应, 然后清空内存放回内存池里面进行反复使用, 这样就大大减少了GC的频率, 保证了生产者的稳定和高效, 性能会大大提高 



```java

public class BufferPool {

    static final String WAIT_TIME_SENSOR_NAME = "bufferpool-wait-time";

    /**
     * 全部内存大小
     */
    private final long totalMemory;

    /**
     * 每一块内存的大小
     */
    private final int poolableSize;

    /**
     *
     */
    private final ReentrantLock lock;

    /**
     * 用于存储空闲的ByteBuffer内存
     */
    private final Deque<ByteBuffer> free;
    
    // 用于添加当内存池内存不足时等待的线程，这里使用Deque保存的话，可以避免线程饥饿问题
    private final Deque<Condition> waiters;
    /** Total available memory is the sum of nonPooledAvailableMemory and the number of byte buffers in free * poolableSize.  */
    /**
     * 未开辟的可用内存，当free列表中没有可用内存时，就可以判断 nonPooledAvailableMemory 是否 > size，如果大于则通过allocateByteBuffer进行开启
     * ，总可用内存 =  nonPooledAvailableMemory + free * poolableSize
     */
    private long nonPooledAvailableMemory;
    private boolean closed;

    /**
     * Create a new buffer pool
     *
     * @param memory The maximum amount of memory that this buffer pool can allocate, 表示当前内存池最大有多少内存可以使用
     * @param poolableSize The buffer size to cache in the free list rather than deallocating
     */
    public BufferPool(long memory, int poolableSize) {
        this.poolableSize = poolableSize;
        this.lock = new ReentrantLock();
        this.free = new ArrayDeque<>();
        this.waiters = new ArrayDeque<>();
        this.totalMemory = memory;
        this.nonPooledAvailableMemory = memory;
        this.closed = false;
    }

    
    //创建一个新的内存，如果没有足够的内存会一直阻塞，直接达到maxTimeToBlockMs
    public ByteBuffer allocate(int size, long maxTimeToBlockMs) throws InterruptedException {
        if (size > this.totalMemory)
            throw new IllegalArgumentException("Attempt to allocate " + size
                                               + " bytes, but there is a hard limit of "
                                               + this.totalMemory
                                               + " on memory allocations.");

        ByteBuffer buffer = null;
        this.lock.lock();

        if (this.closed) {
            this.lock.unlock();
            throw new BusinessException("Producer closed while allocating memory");
        }

        try {
            // check if we have a free buffer of the right size pooled
            //如果请求的内存大小 = 内存块大小，并且free列表不为空则poll出第一个内存
            // 如果空闲队列中没有足够的内存进行分配，则通过nonPooledAvailableMemory来进行分配
            if (size == poolableSize && !this.free.isEmpty())
                return this.free.pollFirst();

            // now check if the request is immediately satisfiable with the
            // memory on hand or if we need to block
            //获取空闲队列的可用内存大小
            int freeListSize = freeSize() * this.poolableSize;

            //如果未开启的内存 + 空闲队列内存大小足够对当前的申请内存进行开启，则直接开辟
            if (this.nonPooledAvailableMemory + freeListSize >= size) {
                // we have enough unallocated or pooled memory to immediately
                // satisfy the request, but need to allocate the buffer
                //循环取出free队列中的内存块并入nonPooledAvailableMemory
                freeUp(size);
                //未开辟的可用内存为 nonPooledAvailableMemory - size
                this.nonPooledAvailableMemory -= size;
            }


            else {
                // we are out of memory and will have to block
                //从当前内存池中拿到的内存
                int accumulated = 0;

                //创建一个新的条件队列
                Condition moreMemory = this.lock.newCondition();
                try {

                    //获取内存的最大阻塞时间
                    long remainingTimeToBlockNs = TimeUnit.MILLISECONDS.toNanos(maxTimeToBlockMs);
                    this.waiters.addLast(moreMemory);
                    // loop over and over until we have a buffer or have reserved
                    // enough memory to allocate one
                    //循环等待从当前内存池到获取内存，直到获取到的内存可以足够满足当前的申请size
                    while (accumulated < size) {
                        long startWaitNs = System.nanoTime();
                        long timeNs;
                        boolean waitingTimeElapsed;
                        try {
                            waitingTimeElapsed = !moreMemory.await(remainingTimeToBlockNs, TimeUnit.NANOSECONDS);
                        } finally {
                            long endWaitNs = System.nanoTime();
                            timeNs = Math.max(0L, endWaitNs - startWaitNs);
                        }

                        if (this.closed)
                            throw new BusinessException("Producer closed while allocating memory");

                        if (waitingTimeElapsed) {
                            throw new BusinessException("Failed to allocate " + size + " bytes within the configured max blocking time "
                                + maxTimeToBlockMs + " ms. Total memory: " + totalMemory() + " bytes. Available memory: " + availableMemory()
                                + " bytes. Poolable size: " + poolableSize() + " bytes");
                        }

                        remainingTimeToBlockNs -= timeNs;

                        // check if we can satisfy this request from the free list,
                        // otherwise allocate memory
                        //如果申请的内存块大小正好等于poolableSize，并且空闲队列也不为空，则直接从队列中虎丘
                        if (accumulated == 0 && size == this.poolableSize && !this.free.isEmpty()) {
                            // just grab a buffer from the free list
                            buffer = this.free.pollFirst();
                            accumulated = size;
                        } else {
                            // we'll need to allocate memory, but we may only get
                            // part of what we need on this iteration

                            //将free队列中buffer循环并入到nonPooledAvailableMemory，直到满足当前内存块的申请
                            freeUp(size - accumulated);

                            //判断nonPooledAvailableMemory是否已经满足当前牛才能块申请了
                            int got = (int) Math.min(size - accumulated, this.nonPooledAvailableMemory);

                            //当前未开辟内存 = nonPooledAvailableMemory - got(即将被分出去的内存)
                            this.nonPooledAvailableMemory -= got;

                            //将accumulated进行累加，到下一次循环时判断是否当前分配的内存是否已经可以满足当前内存块的申请
                            accumulated += got;
                        }
                    }
                    
                    //这里将accumulated主要是为了避免在获取内存超时或者是其他异常情况下需要将现有获取到的内存归还到内存池中
                    // Don't reclaim memory on throwable since nothing was thrown
                    accumulated = 0;
                } finally {
                    
                    //这里分为两种情况，分别是正常获取成功时，以及获取失败时
                    //成功时：this.nonPooledAvailableMemory + 0
                    //失败时：this.nonPooledAvailableMemory + 已经获取到的内存 
                    // When this loop was not able to successfully terminate don't loose available memory
                    this.nonPooledAvailableMemory += accumulated;
                    this.waiters.remove(moreMemory);
                }
            }
        } finally {
            // signal any additional waiters if there is more memory left
            // over for them
            try {
                if (!(this.nonPooledAvailableMemory == 0 && this.free.isEmpty()) && !this.waiters.isEmpty())
                    this.waiters.peekFirst().signal();
            } finally {
                // Another finally... otherwise find bugs complains
                lock.unlock();
            }
        }

        //这里buffer == null 表示free队列为空，表示需要从nonPooledAvailableMemory来进行开辟
        if (buffer == null)
            //开辟一块新的内存进行返回
            return safeAllocateByteBuffer(size);
        else
            return buffer;
    }


    /**
     * Allocate a buffer.  If buffer allocation fails (e.g. because of OOM) then return the size count back to
     * available memory and signal the next waiter if it exists.
     */
    //分配缓冲区。如果缓冲区分配失败(例如因为 OOM) ，那么将大小计数返回到可用内存，并向下一个侍者发出信号(如果存在)。
    private ByteBuffer safeAllocateByteBuffer(int size) {
        boolean error = true;
        try {
            ByteBuffer buffer = allocateByteBuffer(size);
            error = false;
            return buffer;
        } finally {
            if (error) {
                this.lock.lock();
                try {
                    this.nonPooledAvailableMemory += size;
                    if (!this.waiters.isEmpty())
                        this.waiters.peekFirst().signal();
                } finally {
                    this.lock.unlock();
                }
            }
        }
    }

    // Protected for testing.
    protected ByteBuffer allocateByteBuffer(int size) {
        return ByteBuffer.allocate(size);
    }

    /**
     * Attempt to ensure we have at least the requested number of bytes of memory for allocation by deallocating pooled
     * buffers (if needed)
     */
    //如果未开启内存nonPooledAvailableMemory < 当前申请的内存大小
    // 则从free队列的队尾的循环取出buffer直到可以满足当前申请内存的大小
    private void freeUp(int size) {
        while (!this.free.isEmpty() && this.nonPooledAvailableMemory < size)
            this.nonPooledAvailableMemory += this.free.pollLast().capacity();
    }

    /**
     * Return buffers to the pool. If they are of the poolable size add them to the free list, otherwise just mark the
     * memory as free.
     *
     * @param buffer The buffer to return
     * @param size The size of the buffer to mark as deallocated, note that this may be smaller than buffer.capacity
     *             since the buffer may re-allocate itself during in-place compression
     */
    //如果buffer的大小等于 poolableSize,则将该buffer放入到free队列中，否则将nonPooledAvailableMemory + size
    //
    public void deallocate(ByteBuffer buffer, int size) {
        lock.lock();
        try {
            //如果bufferSize == poolableSize 则直接放回队列
            if (size == this.poolableSize && size == buffer.capacity()) {
                buffer.clear();
                this.free.add(buffer);
            } else {

                //否则不对buffer处理，并且将 nonPooledAvailableMemory + size
                this.nonPooledAvailableMemory += size;
            }

            //从等待队列中拿到第一个等待获取内存块的condition，然后释放，这样的作用在于避免现线程饥饿
            Condition moreMem = this.waiters.peekFirst();
            if (moreMem != null)
                moreMem.signal();
        } finally {
            lock.unlock();
        }
    }

    public void deallocate(ByteBuffer buffer) {
        if (buffer != null)
            deallocate(buffer, buffer.capacity());
    }

    /**
     * the total free memory both unallocated and in the free list
     */
    public long availableMemory() {
        lock.lock();
        try {
            return this.nonPooledAvailableMemory + freeSize() * (long) this.poolableSize;
        } finally {
            lock.unlock();
        }
    }

    // Protected for testing.
    protected int freeSize() {
        return this.free.size();
    }

    /**
     * Get the unallocated memory (not in the free list or in use)
     */
    public long unallocatedMemory() {
        lock.lock();
        try {
            return this.nonPooledAvailableMemory;
        } finally {
            lock.unlock();
        }
    }

    /**
     * The number of threads blocked waiting on memory
     */
    public int queued() {
        lock.lock();
        try {
            return this.waiters.size();
        } finally {
            lock.unlock();
        }
    }

    /**
     * The buffer size that will be retained in the free list after use
     */
    public int poolableSize() {
        return this.poolableSize;
    }

    /**
     * The total memory managed by this pool
     */
    public long totalMemory() {
        return this.totalMemory;
    }

    // package-private method used only for testing
    Deque<Condition> waiters() {
        return this.waiters;
    }

    /**
     * Closes the buffer pool. Memory will be prevented from being allocated, but may be deallocated. All allocations
     * awaiting available memory will be notified to abort.
     */
    public void close() {
        this.lock.lock();
        this.closed = true;
        try {
            for (Condition waiter : this.waiters)
                waiter.signal();
        } finally {
            this.lock.unlock();
        }
    }
}
```





## Kafka的消费

### 消费者组

消费者（ Consumer ）负责订阅 Kafka 中的主题（ Topic ），并且从订阅的主题上拉取消息。与其他一些消息中间件不同的是：在 Kafka 的消费理念中还有一层消费组（ Consumer Group) 的概念，每个消费者都有一个对应的消费组。当消息发布到主题后，只会被投递给订阅它的每个消费组中的一个消费者。

![image-20221004183707359](assets/image-20221004183707359.png)



### Kafka的消费流程

![image-20221004214218198](assets/image-20221004214218198.png)



![image-20221004214232238](assets/image-20221004214232238.png)



### 消费者注意点

#### 订阅主题

Kafka可以通过两种方法来订阅主题，分别是subscribe 与 assgin 两种机制, 两种方式

```
public void subscribe(Pattern pattern);
public void subscribe(Collection<String> topics)
public void subscribe(Collection<String> topics, ConsumerRebalanceListener listener)
```

```
public void assign(Collection<TopicPartition> partitions) 
```



####  指定位移消费

Kafka 提供了两种方式用于帮助消费者指定offset的位置，既从何处进行消费

- auto.offset.reset
- seek

**auto.offset.reset**

auto.offset.reset 是当客户端需要订阅broker时设置的参数，当消费者订阅主题时，从哪里进行消费，默认有两个值分别是 earliest 与 latest，如下图：





![image-20221004190523614](assets/image-20221004190523614.png)

- none
  如果没有为消费者找到先前的offset的值,即没有自动维护偏移量,也没有手动维护偏移量,则抛出异常

- earliest
  在各分区下有提交的offset时：从offset处开始消费

  在各分区下无提交的offset时：从头开始消费

- latest
  在各分区下有提交的offset时：从offset处开始消费

  在各分区下无提交的offset时：从最新的数据开始消费




这里需要注意的一个点是如果__consumer_offset 中没有保存 consumer 对应的offset



**seek**

Kafka 提供了一种更细粒度指定offset位移的api，可以让我们从特定的位移处开始拉取消息

```java
//从topic+partition 的offset位置进行消费
public void seek(TopicPartition partition, long offset)
```







#### 消费者拦截器

```java
public interface ConsumerInterceptor<K, V> extends Configurable {

    //在消息消费poll方法调用之前进行调用
    public ConsumerRecords<K, V> onConsume(ConsumerRecords<K, V> records);

   	//在offset提交之前进行调用
    public void onCommit(Map<TopicPartition, OffsetAndMetadata> offsets);

    /**
     * This is called when interceptor is closed
     */
    public void close();
```

Kafka Consumer会在poll（）方法返回之前调用拦截器的onConsume（）方法来对消息进行相应的定制化操作，比如修改返回的消息内容、按照某种规则过滤消息（可能会减少poll（）方法返回的消息的个数〉。如果onConsume（）方法中抛出异常，那么会被捕获并记录到日志中，但是异常不会再向上传递。





### Kafka Rebalance

#### Rebalance 机制

Rebalance就是说如果消费组里的消费者数量有变化或消费的分区数有变化，[kafka](https://so.csdn.net/so/search?q=kafka&spm=1001.2101.3001.7020)会重新分配消费者消费分区的关系。比如consumer group中某个消费者挂了，此时会自动把分配给他的分区交给其他的消费者，如果他又重启了，那么又会把一些分区重新交还给他。

**注意：**

rebalance只针对subscribe这种不指定分区消费的情况，如果通过assign这种消费方式指定了分区，kafka不会进行rebanlance。

如下情况可能会触发消费者rebalance

- 消费组里的consumer增加或减少了
- 动态给topic增加了分区
- 消费组订阅了更多的topic

rebalance过程中，消费者无法从kafka消费消息，这对kafka的TPS会有影响，如果kafka集群内节点较多，比如数百个，那重平衡可能会耗时极多，所以应尽量避免在系统高峰期的重平衡发生，Rebalence 的整个过程如下图所示：

![image-20221004211813202](assets/image-20221004211813202.png)

- heartbeat.interval.ms

  Kafka 消费者和 coordinator 之间的心跳时间，默认 3s。 该条目的值必须小于 session.timeout.ms，也不应该高于  session.timeout.ms 的 1/3

- session.timeout.ms

  Kafka 消费者和 coordinator 之间连接超时时间，默认 45s。超 过该值，该消费者被移除，消费者组执行再平衡

- max.poll.interval.ms 

  消费者处理消息的最大时长，默认是 5 分钟。超过该值，该 消费者被移除，消费者组执行再平衡

- partition.assignment.strategy

  消 费 者 分 区 分 配 策 略 ， 默 认 策 略 是 Range + CooperativeSticky。Kafka 可以同时使用多个分区分配策略。 可 以 选 择 的 策 略 包 括 ： Range 、 RoundRobin 、 Sticky 、CooperativeSticky





#### Rebalance API

Kafka Consumer在订阅某个Topic时提供了ConsumerRebalanceListener监听器，用于在Reblence 发生时进行触发

```
( 1) void onPartitionsRevoked(Collection<TopicPartition> partitions) 
这个方法会在再均衡开始之前和消费者停止读取消息之后被调用。可以通过这个回调方法来处理消费位移的提交， 以此来避免一些不必要的重复消费现象的发生。参数 partitions 表示再均衡前所分配到的分区。

( 2) void onPartitionsAssigned(Collection<TopicPartition> partitions) 
这个方法会在重新分配分区之后和消费者开始读取消费之前被调用 。参数 partitions 表示再均衡后所分配到的分区 。
```





#### Rebalance过程

当有消费者加入消费组时，消费者、消费组及组协调器之间会经历以下几个阶段。

**第一阶段：选择组协调器**

组协调器GroupCoordinator：每个consumer group都会选择一个broker作为自己的组协调器coordinator，负责监控这个消费组里的所有消费者的心跳，以及判断是否宕机，然后开启消费者rebalance。

consumer group中的每个consumer启动时会向kafka集群中的某个节点发送 FindCoordinatorRequest 请求来查找对应的组协调器GroupCoordinator，并跟其建立网络连接。

组协调器选择方式：

consumer消费的offset要提交到__consumer_offsets的哪个分区，这个分区leader对应的broker就是这个consumer group的coordinator

**第二阶段：加入消费组JOIN GROUP**

在成功找到消费组所对应的 GroupCoordinator 之后就进入加入消费组的阶段，在此阶段的消费者会向 GroupCoordinator 发送 JoinGroupRequest 请求，并处理响应。然后GroupCoordinator 从一个consumer group中选择第一个加入group的consumer作为leader(消费组协调器)，把consumer group情况发送给这个leader，接着这个leader会负责制定分区方案。

**第三阶段（ SYNC GROUP)**

consumer leader通过给GroupCoordinator发送SyncGroupRequest，接着GroupCoordinator就把分区方案下发给各个consumer，他们会根据指定分区的leader broker进行网络连接以及消息消费





## Kafka的Offset 存储

Kafka的offset默认是存储在`__consumer_offsets` 的topic中，`__consumer_offsets` 默认会有50个partition，每个partition有三个副本，并且每个partition中保存了消费者对应的offset信息,

 __consumer_offsets 主题里面采用 key 和 value 的方式存储数据。key 是 group.id+topic+分区号，value 就是当前 offset 的值。每隔一段时间，kafka 内部会对这个 topic 进行compact，也就是每个 group.id+topic+分区号就保留最新数据

![image-20221004191658305](assets/image-20221004191658305.png)



#### 创建 `__commit_offset`

当 Kafka 集群中的第一个 Consumer 程序启动时，Kafka 会自动创建位移主题。位移主题就是普通的 Kafka 主题，它也有对应的分区数。但如果是 Kafka 自动创建的，分区数的设定要看 Broker 端参数 offsets.topic.num.partitions 的取值。它的默认值是 50，因此 Kafka 会自动创建一个 50 分区的位移主题。副本数或备份因子是 Broker 端参数 offsets.topic.replication.factor 控制的。默认值是 3。如果 Kafka 日志路径下冒出很多 __consumer_offsets-xxx 这样的目录，这是 Kafka 自动创建的位移主题。


#### 写入offset

Kafka Consumer 提交位移时会写入该主题， Kafka Consumer 提交位移的方式有两种：自动提交位移和手动提交位移。

**自动提交**

Consumer 端有个参数叫 enable.auto.commit，如果值是 true，则 Consumer 在后台定期提交位移，提交间隔由参数 auto.commit.interval.ms 来控制。自动提交位移的优点就是省事，不用操心位移提交的事情，就能保证消息消费不会丢失。但这一点同时也是缺点，丧失了很大的灵活性和可控性，完全没法把控 Consumer 端的位移管理。

- enable.auto.commit**：**是否开启自动提交offset功能，默认是true
-  auto.commit.interval.ms**：**自动提交offset的时间间隔，默认是5s

![image-20221004205535353](assets/image-20221004205535353.png)



**手动提交offset**

手动提交offset的方法有两种：分别是commitSync（同步提交）和commitAsync（异步提交）。两者的相 同点是，都会将本次提交的一批数据最高的偏移量提交；不同点是，同步提交阻塞当前线程，一直到提交成 功，并且会自动失败重试（由不可控因素导致，也会出现提交失败）；而异步提交则没有失败重试机制，故 有可能提交失败。

- commitSync（同步提交）：必须等待offset提交完毕，再去消费下一批数据。
- commitAsync（异步提交） ：发送完提交offset请求后，就开始消费下一批数据了。

![image-20221004205642116](assets/image-20221004205642116.png)





#### 删除offset

Kafka 使用Compact 策略来删除位移主题中的过期消息。对于同一个 Key 的两条消息 M1 和 M2，如果 M1 的发送时间早于 M2，那么 M1 就是过期消息。Compact 的过程就是扫描日志的所有消息，剔除那些过期的消息，然后把剩下的消息整理在一起。图中位移为 0、2 和 3 的消息的 Key 都是 K1。Compact 之后，分区只需要保存位移为 3 的消息，因为它是最新发送的。

Kafka 提供后台线程定期地巡检待 Compact 的主题，看看是否存在满足条件的可删除数据。这个后台线程叫 Log Cleaner。如果位移主题无限膨胀占用过多磁盘空间的问题，可以去检查一下 Log Cleaner 线程的状态，通常都是这个线程挂掉了导致的。

![image-20221004192234965](assets/image-20221004192234965.png)



## Kafka 存储

### 文件存储

topic是逻辑上的概念，而partition是物理上的概念，每个partition对应于一个log文件，该log文件中存储的就是producer生产的数据。Producer生产的数据会被不断追加到该log文件末端，且每条数据都有自己的offset。消费者组中的每个消费者，都会实时记录自己消费到了哪个offset，以便出错恢复时，从上次的位置继续消费

![image-20221004210130175](assets/image-20221004210130175.png)

由于生产者生产的消息会不断追加到log文件末尾，为防止log文件过大导致数据定位效率低下，Kafka采取了分片和索引机制，将每个partition分为多个segment。每个segment对应两个文件——“.index”文件和“.log”文件。这些文件位于一个文件夹下，该文件夹的命名规则为：topic名称+分区序号。例如，first这个topic有三个分区，则其对应的文件夹为first0,first-1,first-2，如下：

```
00000000000000000000.index 
00000000000000000000.log 
00000000000000170410.index 
00000000000000170410.log 
00000000000000239430.index 
00000000000000239430.log
```

index和log文件以当前segment的第一条消息的offset命名。

![图片](assets/640.png)



1. kafka 中消息是以主题 Topic 为基本单位进行归类的，这里的 Topic 是逻辑上的概念，实际上在磁盘存储是根据分区 Partition 存储的, 即每个 Topic 被分成多个 Partition，分区 Partition 的数量可以在主题 Topic 创建的时候进行指定。
2. Partition 分区主要是为了解决 Kafka 存储的水平扩展问题而设计的， 如果一个 Topic 的所有消息都只存储到一个 Kafka Broker上的话， 对于 Kafka 每秒写入几百万消息的高并发系统来说，这个 Broker 肯定会出现瓶颈， 故障时候不好进行恢复，所以 Kafka 将 Topic 的消息划分成多个 Partition， 然后均衡的分布到整个 Kafka Broker 集群中。
3. Partition 分区内每条消息都会被分配一个唯一的消息 id,即我们通常所说的 偏移量 Offset,  因此 kafka 只能保证每个分区内部有序性,并不能保证全局有序性。
4. 然后每个 Partition 分区又被划分成了多个 LogSegment，这是为了防止 Log 日志过大，Kafka 又引入了日志分段(LogSegment)的概念，将 Log 切分为多个 LogSegement，相当于一个巨型文件被平均分割为一些相对较小的文件，这样也便于消息的查找、维护和清理。这样在做历史数据清理的时候，直接删除旧的 LogSegement 文件就可以了。
5. Log 日志在物理上只是以文件夹的形式存储，而每个 LogSegement 对应磁盘上的一个日志文件和两个索引文件，以及可能的其他文件(比如以".snapshot"为后缀的快照索引文件等)



### 定位数据

我们通过上面可以发现Kafka添加数据都是以顺信写的方式添加到LogSegment文件中，并且在添加Message消息时，也会添加相应的索引，用于通过offset来定位Segment文件中的数据，在这里要注意以下几点：

- Kafka在写入LogSegment文件时是以顺序写的方式进行添加的
- Kafka采用了稀疏索引的方式，LogSement 文件每写入4K数据，就会添加一条索引，不过这个索引数据是以offset为维度添加的
- Kafka的LogSegement 文件名称与LogIndex文件名称是相对应的，都是以当前文件中最小的offset来命名的，每当创建一个新的LogSegment文件时就会创建一个LogIndex文件
- 一个Partition对应Segment文件，并且Segment文件大小默认为1G，底层是通过mmap的方式来进行数据写入的
- 在通过offset查找数据时，首先会对segment文件名称或者是index文件名称进行二分查找，通过offset可以定位到具体的Segment文件以及index文件，然后对index文件内部的数据进行二分查找，不过不同于标准的二分查找，由于index文件是以稀疏索引的方式存储的，所以在二分查找时如果查不到对应的数据，会根据查找的类型来返回小于target offset的 offset slot 还是 大于 target offset 的 entry slot



![image-20230301210750273](assets/image-20230301210750273.png)



Log 中写入消息是顺序写入的。**但是只有最后一个 LogSegement 才能执行写入操作**，之前的所有 LogSegement 都不能执行写入操作。为了更好理解这个概念，我们将最后一个 LogSegement 称为**"activeSegement"，即表示当前活跃的日志分段**。随着消息的不断写入，当 activeSegement 满足一定的条件时，就需要创建新的 activeSegement，之后再追加的消息会写入新的 activeSegement。

![image-20230301221512402](assets/image-20230301221512402.png)

为了更高效的进行消息检索，每个 LogSegment 中的日志文件（以“.log”为文件后缀）都有对应的几个索引文件：**偏移量索引文件（以“.index”为文件后缀）、时间戳索引文件（以“.timeindex”为文件后缀）、快照索引文件 （以“.snapshot”为文件后缀）**。其中每个 LogSegment 都有一个 Offset 来作为基准偏移量（baseOffset），用来表示当前 LogSegment 中第一条消息的 Offset。偏移量是一个64位的 Long 长整型数，日志文件和这几个索引文件都是根据基准偏移量（baseOffset）命名的，名称固定为20位数字，没有达到的位数前面用0填充。比如第一个 LogSegment 的基准偏移量为0，对应的日志文件为00000000000000000000.log





![image-20230301210332662](assets/image-20230301210332662.png)



查找message的流程（比如要查找offset为170417的message）：

![image-20221004210640967](assets/image-20221004210640967.png)

> 1. 首先用二分查找确定它是在哪个Segment文件中，其中0000000000000000000.index为最开始的文件，第二个文件为0000000000000170410.index（起始偏移为170410+1 = 170411），而第三个文件为0000000000000239430.index（起始偏移为239430+1 = 239431）。所以这个offset = 170417就落在第二个文件中。其他后续文件可以依此类推，以起始偏移量命名并排列这些文件，然后根据二分查找法就可以快速定位到具体文件位置。
> 2. 用该offset减去索引文件的编号，即170417 - 170410 = 7，也用二分查找法找到索引文件中等于或者小于7的最大的那个编号。可以看出我们能够找到[4，476]这组数据，476即offset=170410 + 4 = 170414的消息在log文件中的偏移量。
> 3. 打开数据文件（0000000000000170410.log），从位置为476的那个地方开始顺序扫描直到找到offset为170417的那条Message

![image-20221004210551588](assets/image-20221004210551588.png)

![image-20221004210849008](assets/image-20221004210849008.png)



### 日志清除

Kafka 将消息存储到磁盘中，随着写入数据不断增加，磁盘占用空间越来越大，为了控制占用空间就需要对消息做一定的清理操作。从上面 Kafka 存储日志结构分析中每一个分区副本（Replica）都对应一个 Log，而 Log 又可以分为多个日志分段（LogSegment），这样就便于 Kafka 对日志的清理操作。

​    Kafka提供了两种日志清理策略：

> 1、日志删除（Log Retention）：按照一定的保留策略直接删除不符合条件的日志分段（LogSegment）。
>
> 2、 日志压缩（Log Compaction）：针对每个消息的key进行整合，对于有相同key的不同value值，只保留最后一个版本。

​    这里我们可以通过 Kafka Broker 端参数 **log.cleanup.policy** 来设置日志清理策略，**默认值为 “delete”，即采用日志删除的清理策略**。如果要采用**日志压缩的清理策略**，就需要将 log.cleanup.policy 设置为 **“compact”**，这样还不够，必须还要将**log.cleaner.****enable（默认值为 true）设为 true**。

如果想要同时支持两种清理策略， 可以直接将 log.cleanup.policy 参数设置为“delete，compact”。

​        

####  日志删除

Kafka 的日志管理器（LogManager）中有一个专门的日志清理任务通过周期性检测和删除不符合条件的日志分段文件（LogSegment），这里我们可以通过 Kafka Broker 端的参数 **log.retention.check.****interval.ms** 来配置，默认值为300000，即5分钟。

 在 Kafka 中一共有3种保留策略：

**基于时间的策略**

日志删除任务会周期检查当前日志文件中是否有保留时间超过设定的阈值**(retentionMs)** 来寻找可删除的日志段文件集合**(****deletableSegments)**。

其中**retentionMs**可以通过 Kafka Broker 端的这几个参数的大小判断的

log.retention.ms > log.retention.minutes > log.retention.hours优先级来设置，**默认情况只会配置 log.retention.hours 参数，值为168即为7天。**

这里需要注意：删除过期的日志段文件，并不是简单的根据该日志段文件的修改时间计算的，而是要根据该日志段中最大的时间戳 largestTimeStamp 来计算的，首先要查询该日志分段所对应的时间戳索引文件，查找该时间戳索引文件的最后一条索引数据，如果时间戳值大于0，则取值，否则才会使用最近修改时间（lastModifiedTime）。

 **【删除步骤】：**

 1、 首先从 Log 对象所维护的日志段的跳跃表中移除要删除的日志段，用来确保已经没有线程来读取这些日志段。   

 2、将日志段所对应的所有文件，包括索引文件都添加上“.deleted”的后缀

 3、最后交给一个以“delete-file”命名的延迟任务来删除这些以“ .deleted ”为后缀的文件。默认1分钟执行一次， 可以通过 file.delete.delay.ms 来配置。

![image-20230301222136986](assets/image-20230301222136986.png)



**基于日志大小的策略**

日志删除任务会周期检查当前日志大小是否超过设定的阈值**(retentionSize)** 来寻找可删除的日志段文件集合**(deletableSegments)**

其中 retentionSize 这里我们可以通过 Kafka Broker 端的参数log.retention.bytes来设置， 默认值为-1，即无穷大。

这里需要注意的是 log.retention.bytes 设置的是Log中所有日志文件的大小，而不是单个日志段的大小。单个日志段可以通过参数 log.segment.bytes 来设置，默认大小为1G

***删除策略***

1、首先计算日志文件的总大小Size和retentionSize的差值，即需要删除的日志总大小。

2、然后从日志文件中的第一个日志段开始进行查找可删除的日志段的文件集合(deletableSegments)

3、找到后就可以进行删除操作了。

![image-20230301222258600](assets/image-20230301222258600.png)



**基于日志的起始偏移量**

该策略判断依据是日志段的下一个日志段的起始偏移量 baseOffset 是否小于等于 logStartOffset，如果是，则可以删除此日志分段。

***【如下图所示 删除步骤】：***

1、首先从头开始遍历每个日志段，日志段 1 的下一个日志分段的起始偏移量为20，小于logStartOffset的大小，将日志段1加入deletableSegments。

2、日志段2的下一个日志偏移量的起始偏移量为35，也小于logStartOffset的大小，将日志分段2页加入deletableSegments。

3、日志段3的下一个日志偏移量的起始偏移量为50，也小于logStartOffset的大小，将日志分段3页加入deletableSegments。

4、日志段4的下一个日志偏移量通过对比后，在logStartOffset的右侧，那么从日志段4开始的所有日志段都不会加入deletableSegments

5、待收集完所有的可删除的日志集合后就可以直接删除了

![image-20230301222444479](assets/image-20230301222444479.png)



#### 日志压缩

**日志压缩 Log Compaction 对于有相同key的不同value值，只保留最后一个版本。**如果应用只关心 key 对应的最新 value 值，则可以开启 Kafka 相应的日志清理功能，Kafka会定期将相同 key 的消息进行合并，只保留最新的 value 值。

​    Log Compaction 可以类比 Redis 中的 RDB 的持久化模式。我们可以想象下，如果每次消息变更都存 Kafka，在某一时刻， Kafka 异常崩溃后，如果想快速恢复，可以直接使用日志压缩策略， 这样在恢复的时候只需要恢复最新的数据即可，这样可以加快恢复速度

![image-20230301222505353](assets/image-20230301222505353.png)



### 磁盘存储

**在 Kafka 中，大量使用了 PageCache**， 这也是 Kafka 能实现高吞吐的重要因素之一， 当一个进程准备读取磁盘上的文件内容时，操作系统会先查看待读取的数据页是否在 PageCache 中，如果命中则直接返回数据，从而避免了对磁盘的 I/O 操作；如果没有命中，操作系统则会向磁盘发起读取请求并将读取的数据页存入 PageCache 中，之后再将数据返回给进程。同样，如果一个进程需要将数据写入磁盘，那么操作系统也会检查数据页是否在页缓存中，如果不存在，则 PageCache 中添加相应的数据页，最后将数据写入对应的数据页。被修改过后的数据页也就变成了脏页，操作系统会在合适的时间把脏页中的数据写入磁盘，以保持数据的一致性。



   **除了消息顺序追加写日志、PageCache以外， kafka 还使用了零拷贝（Zero-Copy）技术来进一步提升系统性能**， 如下图所示：

![image-20230301221724263](assets/image-20230301221724263.png)



### 磁盘写入的整个流程

![图片](assets/640-16776807412123.png)





### Kafka 数据同步

ISR 、AR



## 分区分配策略

博客地址：https://www.cnblogs.com/hzmark/p/sticky_assignor.html

 一个consumer group中有多个consumer，一个topic有多个partition，所以必然会涉及到partition的分配问题，即确定哪个partition由哪个consumer来消费。Kafka提供了3种消费者分区分配策略：RangeAssigor、RoundRobinAssignor、StickyAssignor。

 PartitionAssignor接口用于用户定义实现分区分配算法，以实现Consumer之间的分区分配。消费组的成员订阅它们感兴趣的Topic并将这种订阅关系传递给作为订阅组协调者的Broker。协调者选择其中的一个消费者来执行这个消费组的分区分配并将分配结果转发给消费组内所有的消费者。Kafka默认采用RangeAssignor的分配算法

### RangeAssignor

![image-20221004204740215](assets/image-20221004204740215.png)

RangeAssignor对每个Topic进行独立的分区分配。对于每一个Topic，首先对分区按照分区ID进行排序，然后订阅这个Topic的消费组的消费者再进行排序，之后尽量均衡的将分区分配给消费者。这里只能是尽量均衡，因为分区数可能无法被消费者数量整除，那么有一些消费者就会多分配到一些分区。分配示意图如下：

![img](assets/1460000039010767.png)

分区分配的算法如下：

```java
@Override
public Map<String, List<TopicPartition>> assign(Map<String, Integer> partitionsPerTopic,
                                                Map<String, Subscription> subscriptions) {
    Map<String, List<String>> consumersPerTopic = consumersPerTopic(subscriptions);
    Map<String, List<TopicPartition>> assignment = new HashMap<>();
    for (String memberId : subscriptions.keySet())
        assignment.put(memberId, new ArrayList<TopicPartition>());
    //for循环对订阅的多个topic分别进行处理
    for (Map.Entry<String, List<String>> topicEntry : consumersPerTopic.entrySet()) {
        String topic = topicEntry.getKey();
        List<String> consumersForTopic = topicEntry.getValue();
 
        Integer numPartitionsForTopic = partitionsPerTopic.get(topic);
        if (numPartitionsForTopic == null)
            continue;
        //对消费者进行排序
        Collections.sort(consumersForTopic);
        //计算平均每个消费者分配的分区数
        int numPartitionsPerConsumer = numPartitionsForTopic / consumersForTopic.size();
        //计算平均分配后多出的分区数
        int consumersWithExtraPartition = numPartitionsForTopic % consumersForTopic.size();
 
        List<TopicPartition> partitions = AbstractPartitionAssignor.partitions(topic, numPartitionsForTopic);
        for (int i = 0, n = consumersForTopic.size(); i < n; i++) {
            //计算第i个消费者，分配分区的起始位置
            int start = numPartitionsPerConsumer * i + Math.min(i, consumersWithExtraPartition);
            //计算第i个消费者，分配到的分区数量
            int length = numPartitionsPerConsumer + (i + 1 > consumersWithExtraPartition ? 0 : 1);
            assignment.get(consumersForTopic.get(i)).addAll(partitions.subList(start, start + length));
        }
    }
    return assignment;
}
```

**这种分配方式明显的一个问题是随着消费者订阅的Topic的数量的增加，不均衡的问题会越来越严重，比如上图中4个分区3个消费者的场景，C0会多分配一个分区。如果此时再订阅一个分区数为4的Topic，那么C0又会比C1、C2多分配一个分区，这样C0总共就比C1、C2多分配两个分区了，而且随着Topic的增加，这个情况会越来越严重。分配结果：**

![img](assets/1460000039010772.png)

订阅2个Topic，每个Topic4个分区，共3个Consumer

- **C0：**[T0P0，T0P1，T1P0，T1P1]
- **C1：**[T0P2，T1P2]
- **C2：**[T0P3，T1P3]





### RoundRobinAssignor

RoundRobinAssignor是将所有的分区以及主题展开排序后进行分配，RoundRobinAssignor的分配策略是将消费组内订阅的所有Topic的分区及所有消费者进行排序后尽量均衡的分配（RangeAssignor是针对单个Topic的分区进行排序分配的）。如果消费组内，消费者订阅的Topic列表是相同的（每个消费者都订阅了相同的Topic），那么分配结果是尽量均衡的（消费者之间分配到的分区数的差值不会超过1）。如果订阅的Topic列表是不同的，那么分配结果是不保证“尽量均衡”的，因为某些消费者不参与一些Topic的分配。**RoundRobinAssignor 主要针对的是group内所有消费者订阅的topic是一样的情况下才可以使用的** 

![img](assets/1460000039010773.png)

以上两个topic的情况，相比于之前RangeAssignor的分配策略，可以使分区分配的更均衡。不过考虑这种情况，假设有三个消费者分别为C0、C1、C2，有3个Topic T0、T1、T2，分别拥有1、2、3个分区，并且C0订阅T0，C1订阅T0和T1，C2订阅T0、T1、T2，那么RoundRobinAssignor的分配结果如下：

![img](assets/1460000039010775.png)

看上去分配已经尽量的保证均衡了，不过可以发现C2承担了4个分区的消费而C1订阅了T1，是不是把T1P1交给C1消费能更加的均衡呢？



### StickyAssignor

StickyAssignor分区分配算法，目的是在执行一次新的分配时，能在上一次分配的结果的基础上，尽量少的调整分区分配的变动，节省因分区分配变化带来的开销。Sticky是“粘性的”，可以理解为分配结果是带“粘性的”——每一次分配变更相对上一次分配做最少的变动。其目标有两点：

- 分区的分配尽量的均衡。
- 每一次重分配的结果尽量与上一次分配结果保持一致。

当这两个目标发生冲突时，优先保证第一个目标。第一个目标是每个分配算法都尽量尝试去完成的，而第二个目标才真正体现出StickyAssignor特性的。

StickyAssignor算法比较复杂，下面举例来说明分配的效果（对比RoundRobinAssignor），前提条件：

- 有4个Topic：T0、T1、T2、T3，每个Topic有2个分区。
- 有3个Consumer：C0、C1、C2，所有Consumer都订阅了这4个分区。

![img](assets/1460000039010776.png)

上面红色的箭头代表的是有变动的分区分配，可以看出，StickyAssignor的分配策略，变动较小。



## 幂等与事务

### 幂等

**幂等性**就是指Producer不论向Broker发送多少次重复数据，Broker端都只会持久化一条，保证了不重复。Kafka幂等采用了 Exactly Once理念如下：

- **精确一次（Exactly Once）= 幂等性 + 至少一次（ ack=-1 + 分区副本数>=2 + ISR最小副本数量>=2） 。**

**重复数据的判断标准**：具有相同主键<PID, Partition, SeqNumber> 的消息提交时，Broker 只会持久化一条。其 中PID是Kafka每次重启都会分配一个新的；Partition 表示分区号；Sequence Number是单调自增的。

所以幂等性只能保证的是在单分区单会话内不重复, 可以通过参数开启。

- enable.idempotence 默认为 true，false 关闭

![image-20221004214659652](assets/image-20221004214659652.png)



### 事务

```
// 1 初始化事务 
void initTransactions(); 
 
// 2 开启事务 
void beginTransaction() throws ProducerFencedException; 
 
// 3 在事务内提交已经消费的偏移量（主要用于消费者） 
void sendOffsetsToTransaction(Map<TopicPartition, OffsetAndMetadata> offsets, 
 String consumerGroupId) throws 
ProducerFencedException; 
 
// 4 提交事务 
void commitTransaction() throws ProducerFencedException; 
 
// 5 放弃事务（类似于回滚事务的操作） 
void abortTransaction() throws ProducerFencedException;
```

![image-20221004214955550](assets/image-20221004214955550.png)



## 消息堆积

### 监控

通过kafka edger 监控每个partition 的lag 

![image-20221004215158563](assets/image-20221004215158563.png)

### 解决方案

### 案例

假如现在有一个在线的业务，突然量大了起来，消费端或者说下游系统突然处理不过来了，MQ出现了大量的消息堆积，业务写入MQ异常，有什么办法可以相对平缓得解决呢？



### 解决思路

#### 从生产者端解决

一般我们的系统容量或者处理能力都是规划好的，出现消息堆积的情况，大部分是由于流量暴增引起，这个时候可以考虑控制生产者的速率，对前端机器流量进行限速限流。

#### 从消费者端解决

1、consumer 消费能力不足，partition != 添加consumer 实例就需要进行添加 consumer

2、在处理消息队列消息无序的情况下，consumer 消费能力不足，增大partition 的数量以及增加consumer实例

3、consumer 程序出现问题，修复consumer程序，并且将原topic中的数据通过程序导出到新的topic中，然后增加partition 比如以前是5个partition，现在就增加到20个，consumer程序修复好后对新的topic中的数据进行消费



## Controller选举

集群控制器组件（BrokerController）：

它是 Kafka 的核心组件。它的主要作用是在 ZooKeeper 的帮助下管理和协调整个 Kafka 集群，集群中的每个 broker 都可以称为 controller，但是在 Kafka 集群启动后，只有一个 broker 会成为 Controller 。
Controller Broker的主要职责有很多，主要是一些管理行为，主要包括以下几个方面：

• 创建、删除主题，增加分区并分配leader分区
• 集群Broker管理（新增 Broker、Broker 主动关闭、Broker 故障)
• preferred leader选举

分区重分配Kafka是基于zookeeper的，controller的选择也是在zookeeper上完成的。
Kafka 当前选举控制器的规则是：Kafka 集群中第一个启动的 broker 通过在 ZooKeeper 里创建一个临时节点 /controller 让自己成为 controller 控制器。其他 broker 在启动时也会尝试创建这个节点，但是由于这个节点已存在，所以后面想要创建 /controller 节点时就会收到一个 节点已存在 的异常。然后其他 broker 会在这个控制器上注册一个 ZooKeeper 的 watch 对象，/controller节点发生变化时，其他 broker 就会收到节点变更通知。这种方式可以确保只有一个控制器存在。那么只有单独的节点一定是有个问题的，那就是单点问题。

如果控制器关闭或者与 ZooKeeper 断开链接，ZooKeeper 上的临时节点就会消失。集群中的其他节点收到 watch 对象发送控制器下线的消息后，其他 broker 节点都会尝试让自己去成为新的控制器。其他节点的创建规则和第一个节点的创建原则一致，都是第一个在 ZooKeeper 里成功创建控制器节点的 broker 会成为新的控制器，那么其他节点就会收到节点已存在的异常，然后在新的控制器节点上再次创建 watch 对象进行监听。
