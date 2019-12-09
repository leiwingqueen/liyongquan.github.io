---
layout: post
title:  "otter select源码分析"
description: otter select源码分析
date:   2019-12-5 09:25:00 +000
categories: otter 数据一致性
tags: otter 数据一致性
---

### 仲裁者

setl模型都使用了*热备机制*。为了保证高可用和数据一致性，otter在每个时刻只允许一个实例运行，因此需要zookeeper来进行仲裁。

`MainstemMonitor`

```java
/**
     * 检查当前的状态
     */
    public boolean check() {
        String path = StagePathUtils.getMainStem(getPipelineId());
        try {
            byte[] bytes = zookeeper.readData(path);
            Long nid = ArbitrateConfigUtils.getCurrentNid();
            MainStemEventData eventData = JsonUtils.unmarshalFromByte(bytes, MainStemEventData.class);
            activeData = eventData;// 更新下为最新值
            // 检查下nid是否为自己
            boolean result = nid.equals(eventData.getNid());
            if (!result) {
                logger.warn("mainstem is running in node[{}] , but not in node[{}]", eventData.getNid(), nid);
            }
            return result;
        } catch (ZkNoNodeException e) {
            logger.warn("mainstem is not run any in node");
            return false;
        } catch (ZkInterruptedException e) {
            logger.warn("mainstem check is interrupt");
            Thread.interrupted();// 清除interrupt标记
            return check();
        } catch (ZkException e) {
            logger.warn("mainstem check is failed");
            return false;
        }
    }
```

在zk的路径下注册`${pipelineId}/mainstem`当前的活动的实例信息，判断当前的活动的实例是否自己，`boolean result = nid.equals(eventData.getNid());`。

### CanalEmbedSelector

`CanalEmbedSelector`是otter内置canal客户端

```java
public Message<EventData> selector() throws InterruptedException {
        int emptyTimes = 0;
        com.alibaba.otter.canal.protocol.Message message = null;
        if (batchTimeout < 0) {// 进行轮询处理
            while (running) {
                message = canalServer.getWithoutAck(clientIdentity, batchSize);
                if (message == null || message.getId() == -1L) { // 代表没数据
                    applyWait(emptyTimes++);
                } else {
                    break;
                }
            }
            if (!running) {
                throw new InterruptedException();
            }
        ...

        List<Entry> entries = null;
        if (message.isRaw()) {
            entries = new ArrayList<CanalEntry.Entry>(message.getRawEntries().size());
            for (ByteString entry : message.getRawEntries()) {
                try {
                    entries.add(CanalEntry.Entry.parseFrom(entry));
                } catch (InvalidProtocolBufferException e) {
                    throw new SelectException(e);
                }
            }
        } else {
            entries = message.getEntries();
        }

        List<EventData> eventDatas = messageParser.parse(pipelineId, entries); // 过滤事务头/尾和回环数据
        Message<EventData> result = new Message<EventData>(message.getId(), eventDatas);
        // 更新一下最后的entry时间，包括被过滤的数据
        if (!CollectionUtils.isEmpty(entries)) {
            long lastEntryTime = entries.get(entries.size() - 1).getHeader().getExecuteTime();
            if (lastEntryTime > 0) {// oracle的时间可能为0
                this.lastEntryTime = lastEntryTime;
            }
        }
    ...
        return result;
    }
```

核心流程有3步

- 从canal server批量获取binlog(无需ack)
- 过滤事务头/尾和回环数据
- 更新最后的entry time

`MessageParser`为otter的binlog解析

```java
 /**
     * 将对应canal送出来的Entry对象解析为otter使用的内部对象
     * 
     * <pre>
     * 需要处理数据过滤：
     * 1. Transaction Begin/End过滤
     * 2. retl.retl_client/retl.retl_mark 回环标记处理以及后续的回环数据过滤
     * 3. retl.xdual canal心跳表数据过滤
     * </pre>
     */
    public List<EventData> parse(Long pipelineId, List<Entry> datas) throws SelectException {
```

核心的binlog解析都在这里实现。主要包含下面两个核心逻辑

- Transaction Begin/End 处理
- 回环标记处理

回环表retl_mark

```sql
1. 创建database retl
*/
CREATE DATABASE retl;
/* 2. 用户授权 给同步用户授权 */
CREATE USER retl@'%' IDENTIFIED BY 'retl';
GRANT USAGE ON *.* TO `retl`@'%';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO `retl`@'%';
GRANT SELECT, INSERT, UPDATE, DELETE, EXECUTE ON `retl`.* TO `retl`@'%';
/* 业务表授权，这里可以限定只授权同步业务的表 */
GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO `retl`@'%';
/* 3. 创建系统表 */
USE retl;
DROP TABLE IF EXISTS retl.retl_buffer;
DROP TABLE IF EXISTS retl.retl_mark;
DROP TABLE IF EXISTS retl.xdual;
CREATE TABLE retl_buffer
(    
    ID BIGINT(20) AUTO_INCREMENT,
    TABLE_ID INT(11) NOT NULL,
    FULL_NAME varchar(512),
    TYPE CHAR(1) NOT NULL,
    PK_DATA VARCHAR(256) NOT NULL,
    GMT_CREATE TIMESTAMP NOT NULL,
    GMT_MODIFIED TIMESTAMP NOT NULL,
    CONSTRAINT RETL_BUFFER_ID PRIMARY KEY (ID) 
)  ENGINE=InnoDB DEFAULT CHARSET=utf8;
CREATE TABLE retl_mark
(    
    ID BIGINT AUTO_INCREMENT,
    CHANNEL_ID INT(11),
    CHANNEL_INFO varchar(128),
    CONSTRAINT RETL_MARK_ID PRIMARY KEY (ID) 
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
CREATE TABLE xdual (
ID BIGINT(20) NOT NULL AUTO_INCREMENT,
X timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
PRIMARY KEY (ID)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
/* 4. 插入初始化数据 */
INSERT INTO retl.xdual(id, x) VALUES (1,now()) ON DUPLICATE KEY UPDATE x = now();
```

### OtterSelector

otter同步数据获取

```java
/**
 * otter同步数据获取
 * 
 * @author jianghang 2012-7-31 下午02:30:33
 */
public interface OtterSelector<T> {

    /**
     * 启动
     */
    public void start();

    /**
     * 是否启动
     */
    public boolean isStart();

    /**
     * 关闭
     */
    public void stop();

    /**
     * 获取一批待处理的数据
     */
    public Message<T> selector() throws InterruptedException;

    /**
     * 返回未被ack的数据
     */
    public List<Long> unAckBatchs();

    /**
     * 反馈一批数据处理失败，需要下次重新被处理
     */
    public void rollback(Long batchId);

    /**
     * 反馈所有的batch数据需要被重新处理
     */
    public void rollback();

    /**
     * 反馈一批数据处理完成
     */
    public void ack(Long batchId);

    /**
     * 返回最后一次entry数据的时间戳
     */
    public Long lastEntryTime();
}
```

这里只需要关注3个接口

- selector
- ack
- rollback

### SelectTask

`SelectTask`启动

```java
    private void startup() throws InterruptedException {
        try {
            arbitrateEventService.mainStemEvent().await(pipelineId);
        } catch (Throwable e) {
            if (isInterrupt(e)) {
                logger.info("INFO ## this node is interrupt", e);
            } else {
                logger.warn("WARN ## this node is crashed.", e);
            }
            arbitrateEventService.mainStemEvent().release(pipelineId);
            return;
        }

        executor = Executors.newFixedThreadPool(2); // 启动两个线程
        // 启动selector
        otterSelector = otterSelectorFactory.getSelector(pipelineId); // 获取对应的selector
        otterSelector.start();

        canStartSelector.set(false);// 初始化为false
        startProcessTermin();
        startProcessSelect();

        isStart = true;
    }
```

主要实现3步

- 启动selector
- startProcessTermin
- startProcessSelect

#### 启动selector

调用`OtterSelector.start()`方法，构建一个CanalSever并启动。

#### startProcessTermin

处理结束信号。ack或者rollback。

#### startProcessSelect

执行数据分发工作。

```java
private void processSelect() {
        while (running) {
            //获取数据源的binlog数据   
			Message gotMessage = otterSelector.selector();
            ...
                final BatchTermin batchTermin = new BatchTermin(message.getId(), etlEventData.getProcessId());
                batchBuffer.put(batchTermin); // 添加到待响应的buffer列表
```

### 总结

`SelectTask`是整个SETL模型的第一步，这里先做一个简单分析。

### 参考文献

[分布式数据库同步系统Otter](https://www.jianshu.com/p/bce6fb8862cb)











