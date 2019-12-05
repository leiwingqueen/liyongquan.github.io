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













