---
title: 分布式唯一ID生成算法
categories: [雪花算发,唯一ID]
tags: [分布式]
author: [xiao_e]
---

## 雪花算法

雪花算法是Twitter设计的一个分布式ID生成算法，应用于每篇推文的ID。

### 格式定义

雪花算法把一个`64位的二进制`划分位3部分，如下图所示：

![img](https://cdn.jsdelivr.net/gh/cxyxq/images/snowflake-format.png)

解释：
1. 第一位弃用表示符号位，id为正数
1. 前面的41位表示当前时间戳，最多表示69年的时间：
2. 中间的10位表示机器ID,不能重复，最大表示1024台机器ID。如果有多个集群，也可以把5位表示集群，5位表示机器id
3. 最后的12位表示序列号，同一毫秒最大表示4096个序列号

### 使用年限优化

> 1. 如果41位全部表示时间戳，那么只能使用**69年**的时间即到2039年, 因为基准时间戳为：0
>> (1L<<41) / (1000L * 3600 * 24 * 365) ≈ 69.77
> 
> 2. 设置一个新的**基准时间戳**，比如：1451577600(2016-01-01)， 虽然41位最大能表示69年，但是要到2085时，减去基准时间戳后才能达到(1L<<41)的最大值。
> 使用年限就可以延长到 2016 + 69 = 2085， 多使用46年
>> 当前时间戳 1714982186 - 1451577600 = 263404586
{: .prompt-tip}

## 百度 uid-generator

由百度基于雪花算法设计的一个分布式id生成器，它的空间划分是这样的：

![Snowflake](https://cdn.jsdelivr.net/gh/cxyxq/images/snowflake-baidu-uid-snowflake.png)

- sign(1bit)
  固定1bit符号标识，即生成的UID为正数。
- delta seconds (28 bits)
  当前时间，相对于时间基点"2016-05-20"的增量值，单位：秒，最多可支持约8.7年
- worker id (22 bits)
  机器id，最多可支持约420w次机器启动。内置实现为在启动时由数据库分配，默认分配策略为用后即弃，后续可提供复用策略。
- sequence (13 bits)
  每秒下的并发序列，13 bits可支持每秒8192个并发。

### 分配 worker id

uid-generator使用`database`来分配worker id，每次server启动时，都会向数据库获取一个新的worker id。如果分配`22 bits` worker id, 则最多支持约420w次启动。

```sql
DROP TABLE IF EXISTS WORKER_NODE;
CREATE TABLE WORKER_NODE
(
ID BIGINT NOT NULL AUTO_INCREMENT COMMENT 'auto increment id',
HOST_NAME VARCHAR(64) NOT NULL COMMENT 'host name',
PORT VARCHAR(64) NOT NULL COMMENT 'port',
TYPE INT NOT NULL COMMENT 'node type: ACTUAL or CONTAINER',
LAUNCH_DATE DATE NOT NULL COMMENT 'launch date',
MODIFIED TIMESTAMP NOT NULL COMMENT 'modified time',
CREATED TIMESTAMP NOT NULL COMMENT 'created time',
PRIMARY KEY(ID)
)
 COMMENT='DB WorkerID Assigner for UID Generator',ENGINE = INNODB;
```
{: file="WORKER_NODE"}

启动时获取worker id

```java
@Transactional
public long assignWorkerId() {
  // build worker node entity
  WorkerNodeEntity workerNodeEntity = buildWorkerNode();

  // add worker node for new (ignore the same IP + PORT)
  workerNodeDAO.addWorkerNode(workerNodeEntity);
  LOGGER.info("Add worker node:" + workerNodeEntity);

  return workerNodeEntity.getId();
}
```
{: file="DisposableWorkerIdAssigner"}

插入WORKER_NODE获取新worker id
```sql
INSERT INTO WORKER_NODE (HOST_NAME, PORT, TYPE, LAUNCH_DATE, MODIFIED, CREATED)
VALUES ( #{hostName}, #{port}, #{TYPE}, #{launchDate}, NOW(),NOW())
```
{: file="addWorkerNode"}

### DefaultUidGenerator
默认实现的uid生成算法如下 (部分核心逻辑)：

启动时初始化 `timeBits，workerBits，seqBits` 和 `workerId`
```java
@Override
    public void afterPropertiesSet() throws Exception {
        // 分配 timeBits，workerBits，seqBits
        bitsAllocator = new BitsAllocator(timeBits, workerBits, seqBits);

        // 初始化 worker id
        workerId = workerIdAssigner.assignWorkerId();
        if (workerId > bitsAllocator.getMaxWorkerId()) {
            throw new RuntimeException("Worker id " + workerId + " exceeds the max " + bitsAllocator.getMaxWorkerId());
        }

        LOGGER.info("Initialized bits(1, {}, {}, {}) for workerID:{}", timeBits, workerBits, seqBits, workerId);
    }
```
{: file="DefaultUidGenerator#afterPropertiesSet"}

获取UID
```java
protected synchronized long nextId() {
        long currentSecond = getCurrentSecond();

        // 发现时钟回退,抛出错误,不生成uid
        if (currentSecond < lastSecond) {
            long refusedSeconds = lastSecond - currentSecond;
            throw new UidGenerateException("Clock moved backwards. Refusing for %d seconds", refusedSeconds);
        }

        // 当前秒数等于最后一次生成uid时的秒数lastSecond
  			// 同一秒可是支持8192个序号，直接sequence+1
        if (currentSecond == lastSecond) {
            sequence = (sequence + 1) & bitsAllocator.getMaxSequence();
            // !!! 超过了当前秒数最大的sequence，则需要等待下一秒的到来！！！
            if (sequence == 0) {
                currentSecond = getNextSecond(lastSecond);
            }

        // 当前时间已经超过了lastSecond，直接从0开始计数
        // 比如最后一次是10秒之前生成了id， 现在是10秒以后了，重新计数
        } else {
            sequence = 0L;
        }

        lastSecond = currentSecond; //更新最后一次生成ui的秒数

        /**
         * 生成UID，进行位运算
         *   (deltaSeconds << timestampShift) | (workerId << workerIdShift) | sequence;
         *
         * 其中的偏移量：
         *  timestampShift = workerIdBits + sequenceBits;//35 = 22 + 13
         *  workerIdShift = sequenceBits;//13
         *
         * long epochSeconds = TimeUnit.MILLISECONDS.toSeconds(1463673600000L); //2016-05-20 00:00:00
         *
         * 例如：currentSecond=1714902489 workerId=1024 sequence=8
         * ((1714902489 - 1463673600) << 35) | (workerId << 13) | sequence;
         * 得到如下二进制：1110111110010111001011011001000
         */
        return bitsAllocator.allocate(currentSecond - epochSeconds, workerId, sequence);
    }
```
{: file="getUID"}

### CachedUidGenerator
`CachedUidGenerator`使用`提前生成uid`的方式来进行优化性能，提前生成的uid缓存到`RingBuffer`环形数组，数组每个元素成为一个`slot`。在提前生成uid时它支持`初始化预填充`，`即时填充`，`周期填充`三种填充方式。

> `RingBuffer`的容量默认为Snowflake算法中`sequence`最大值`8192`，且必须为`2的幂次方`。可通过`boostPower`来配置进行扩容，以提高RingBuffer 读写吞吐量。
> 
> 默认 boostPower = 3， 原bufferSize=8192, 扩容后bufferSize= 8192 << 3 = 65536
{: .prompt-tip}

#### RingBuffer设计思想：
`Tail`指针、`Cursor`指针用于环形数组上读写`slot`：

##### Tail指针
表示Producer生产的最大序号(此序号从0开始，持续递增)。Tail不能超过Cursor，即生产者不能覆盖未消费的slot。当Tail已赶上curosr，此时可通过rejectedPutBufferHandler指定PutRejectPolicy

##### Cursor指针
表示Consumer消费到的最小序号(序号序列与Producer序列相同)。Cursor不能超过Tail，即不能消费未生产的slot。当Cursor已赶上tail，此时可通过rejectedTakeBufferHandler指定TakeRejectPolicy

![RingBuffer](https://cdn.jsdelivr.net/gh/cxyxq/images/snowflake-baidu-uid-ringbuffer.png)

CachedUidGenerator采用了双RingBuffer，Uid-RingBuffer用于存储Uid、Flag-RingBuffer用于存储Uid状态(是否可填充、是否可消费)

#### RingBuffer填充时机

- 初始化预填充
  RingBuffer初始化时，预先填充满整个RingBuffer.
- 即时填充
  Take消费时，即时检查剩余可用slot量(`tail` - `cursor`)，如小于设定阈值，则补全空闲slots。阈值可通过`paddingFactor`来进行配置，请参考Quick Start中CachedUidGenerator配置
- 周期填充
  通过Schedule线程，定时补全空闲slots。可通过`scheduleInterval`配置，以应用定时填充功能，并指定Schedule时间间隔

#### RingBuffer实现细节

##### 初始化
CachedUidGenerator启动时进行初始化` wokerId` 和 `RingBuffer`：
```java
    public void afterPropertiesSet() throws Exception {
        // 初始化workerId 和 bits位数, 复用的 DefaultUidGenerator 的逻辑
        super.afterPropertiesSet();
        
        //初始化 RingBuffer 和 RingBuffer填充executor
        this.initRingBuffer();
        LOGGER.info("Initialized RingBuffer successfully.");
    }
    
    //初始化 RingBuffer
    private void initRingBuffer() {
        // initialize RingBuffer
        int bufferSize = ((int) bitsAllocator.getMaxSequence() + 1) << boostPower;
        this.ringBuffer = new RingBuffer(bufferSize, paddingFactor);
        LOGGER.info("Initialized ring buffer size:{}, paddingFactor:{}", bufferSize, paddingFactor);

        // initialize RingBufferPaddingExecutor
        boolean usingSchedule = (scheduleInterval != null);
        this.bufferPaddingExecutor = new BufferPaddingExecutor(ringBuffer, this::nextIdsForOneSecond, usingSchedule);
        if (usingSchedule) {
            bufferPaddingExecutor.setScheduleInterval(scheduleInterval);
        }
        
        LOGGER.info("Initialized BufferPaddingExecutor. Using schdule:{}, interval:{}", usingSchedule, scheduleInterval);
        
        // set rejected put/take handle policy
        this.ringBuffer.setBufferPaddingExecutor(bufferPaddingExecutor);
        if (rejectedPutBufferHandler != null) {
            this.ringBuffer.setRejectedPutHandler(rejectedPutBufferHandler);
        }
        if (rejectedTakeBufferHandler != null) {
            this.ringBuffer.setRejectedTakeHandler(rejectedTakeBufferHandler);
        }
        
        // fill in all slots of the RingBuffer
        bufferPaddingExecutor.paddingBuffer();
        
        // start buffer padding threads
        bufferPaddingExecutor.start();
    }
```
{: file="CachedUidGenerator extends DefaultUidGenerator implements DisposableBean"}

##### 获取UID

```java
public long getUID() {
        try {
            return ringBuffer.take(); .//从ringBuffer获取
        } catch (Exception e) {
            LOGGER.error("Generate unique id exception. ", e);
            throw new UidGenerateException(e);
        }
    }
```

从ringBuffer获取uid
```java
/**
     * Take an UID of the ring at the next cursor, this is a lock free operation by using atomic cursor<p>
     * 
     * Before getting the UID, we also check whether reach the padding threshold, 
     * the padding buffer operation will be triggered in another thread<br>
     * If there is no more available UID to be taken, the specified {@link RejectedTakeBufferHandler} will be applied<br>
     * 
     * @return UID
     * @throws IllegalStateException if the cursor moved back
     */
    public long take() {
        // spin get next available cursor
        long currentCursor = cursor.get();
        long nextCursor = cursor.updateAndGet(old -> old == tail.get() ? old : old + 1);

        // check for safety consideration, it never occurs
        Assert.isTrue(nextCursor >= currentCursor, "Curosr can't move back");

        // trigger padding in an async-mode if reach the threshold
        long currentTail = tail.get();
        if (currentTail - nextCursor < paddingThreshold) {
            LOGGER.info("Reach the padding threshold:{}. tail:{}, cursor:{}, rest:{}", paddingThreshold, currentTail,
                    nextCursor, currentTail - nextCursor);
            bufferPaddingExecutor.asyncPadding(); //触发异步填充
        }

        // cursor catch the tail, means that there is no more available UID to take
        if (nextCursor == currentCursor) {
            rejectedTakeHandler.rejectTakeBuffer(this);
        }

        // 1. check next slot flag is CAN_TAKE_FLAG
        int nextCursorIndex = calSlotIndex(nextCursor);
        Assert.isTrue(flags[nextCursorIndex].get() == CAN_TAKE_FLAG, "Curosr not in can take status");

        // 2. get UID from next slot
        // 3. set next slot flag as CAN_PUT_FLAG.
        long uid = slots[nextCursorIndex]; //！！！获取到了uid！！！
        flags[nextCursorIndex].set(CAN_PUT_FLAG);

        // Note that: Step 2,3 can not swap. If we set flag before get value of slot, the producer may overwrite the
        // slot with a new UID, and this may cause the consumer take the UID twice after walk a round the ring
        return uid;
    }
```
{: file="RingBuffer"}

触发异步填充
```java
public void paddingBuffer() {
        LOGGER.info("Ready to padding buffer lastSecond:{}. {}", lastSecond.get(), ringBuffer);

        // is still running
        if (!running.compareAndSet(false, true)) {
            LOGGER.info("Padding buffer is still running. {}", ringBuffer);
            return;
        }

        // fill the rest slots until to catch the cursor
        boolean isFullRingBuffer = false;
        while (!isFullRingBuffer) {
            List<Long> uidList = uidProvider.provide(lastSecond.incrementAndGet()); //CachedUidGenerator#nextIdsForOneSecond 获取未来的uidList进行填充
            for (Long uid : uidList) {
                isFullRingBuffer = !ringBuffer.put(uid);
                if (isFullRingBuffer) {
                    break;
                }
            }
        }

        // not running now
        running.compareAndSet(true, false);
        LOGGER.info("End to padding buffer lastSecond:{}. {}", lastSecond.get(), ringBuffer);
    }
```
{: file="bufferPaddingExecutor.asyncPadding"}

获取下一秒内所有的ids
```java
protected List<Long> nextIdsForOneSecond(long currentSecond) {
        // Initialize result list size of (max sequence + 1)
        int listSize = (int) bitsAllocator.getMaxSequence() + 1;
        List<Long> uidList = new ArrayList<>(listSize);

        // Allocate the first sequence of the second, the others can be calculated with the offset
        long firstSeqUid = bitsAllocator.allocate(currentSecond - epochSeconds, workerId, 0L);
        for (int offset = 0; offset < listSize; offset++) {
            uidList.add(firstSeqUid + offset);
        }

        return uidList;
}
```
{: file="CachedUidGenerator#nextIdsForOneSecond"}


## 美团 Leaf

## worker id 分配策略

### 手动分配
如果集群规模小，且不常发生变化，可以直接手动给每个server指定workerId

### 动态分配
当集群规模较大时，就要使用自动分配的方式了，一般有如下方式：

#### Zookeeper
每个server启动后，把自己的 `ip:port` 注册到zk上，zk返回一个创建的序号，使用序号作为server的workerId。

```bin
# create the persistent-sequential node
[zkshell: 9] create -s /persistent_sequential_node mydata
	Created /persistent_sequential_node0000000176

# create the ephemeral-sequential_node
[zkshell: 10] create -s -e /ephemeral_sequential_node mydata
	Created /ephemeral_sequential_node0000000174
```

#### database
上面百度 uid-generator使用的就是数据库分配的方式。

### K8S部署后IP不固定的问题

## 时钟回退问题

### 什么是时钟回退

### 如何解决回退问题
#### baidu uid-generator
- 默认实现方式没有解决回退问题，当发生了回退时，直接报错了，宁可不生成也不返回一个可能重复的uid。

- 使用RingBuffer的方式没有时钟回退问题：
	- lastSecond不会回退

		`lastSecond`在`initRingBuffer`时进行第一次赋值后，后续再进行填充buffer时每次都是在`lastSecond`的基础上进行+1来获取uidList。

>  RingBuffer就算发生了时钟回退，也不影响`lastSecond`的值。就算此时当前机器发生了重启，且重启后时钟依然回退，也不影响，因为：workerId 变了！！！厉害～～～
{: .prompt-info}

#### 美团 Leaf

Leaf当发现时钟回退时，会等待5ms钟，尽量让时间追赶上来，如果还是未追赶上，则进行报警。
```java
//发生了回拨，此刻时间小于上次发号时间
 if (timestamp < lastTimestamp) {
   long offset = lastTimestamp - timestamp;
   if (offset <= 5) {
     try {
       //时间偏差大小小于5ms，则等待两倍时间
       wait(offset << 1);//wait
       timestamp = timeGen();
       if (timestamp < lastTimestamp) {
         //还是小于，抛异常并上报
         throwClockBackwardsEx(timestamp);
       }    
     } catch (InterruptedException e) {  
       throw  e;
     }
   } else {
     //throw
     throwClockBackwardsEx(timestamp);
   }
  }
 //分配ID       
```



### 参考文章
1. [百度 UidGenerator](https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md)
2. [美团 Leaf](https://tech.meituan.com/2017/04/21/mt-leaf.html)