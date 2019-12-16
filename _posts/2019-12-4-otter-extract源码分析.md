---
layout: post
title:  "otter extract源码分析"
description: otter extract源码分析
date:   2019-12-4 09:26:00 +000
categories: otter 数据一致性
tags: otter 数据一致性
---

### 一、前言

SETL是otter的核心组件，我们将逐个模块分析和了解otter的实现机制。我们先对E(extract)的组件做分析。

### 二、核心组件

`ExtractTask` 是extract的工作线程。

![ExtractTask类图](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20191204093804.png)

`GlobalTask` 是SETL的父线程。封装了工作线程需要的一些公共属性。

```java
public abstract class GlobalTask extends Thread {

    protected final Logger              logger  = LoggerFactory.getLogger(this.getClass());
    //控制线程运行状态
    protected volatile boolean          running = true;
    //从源端到目标端的整个过程描述，主要由一些同步映射过程组成。简单来说，一个同步规则就是一个pipeline
    protected Pipeline                  pipeline;
    //pipeline对应的唯一ID
    protected Long                      pipelineId;
    //仲裁事件相关控制(HA)
    protected ArbitrateEventService     arbitrateEventService;
    //行数据(binlog)?
    protected RowDataPipeDelegate       rowDataPipeDelegate;
    //线程池
    protected ExecutorService           executorService;
    //客户端配置?
    protected ConfigClientService       configClientService;
    protected StageAggregationCollector stageAggregationCollector;
    protected Map<Long, Future>         pendingFuture;
```

### 三、源码分析

`ExtractTask`。extrack任务的核心流程，我这里把非核心的代码逻辑已经去掉，只保留最核心的业务逻辑。

```java
public void run() {
...
        while (running) {
            try {
                final EtlEventData etlEventData = arbitrateEventService.extractEvent().await(pipelineId);
                Runnable task = new Runnable() {

                    public void run() {
                        ...
                        try {
                            pipeline = configClientService.findPipeline(pipelineId);
                            List<PipeKey> keys = (List<PipeKey>) etlEventData.getDesc();
                            long nextNodeId = etlEventData.getNextNid();
                            //select过程中的数据会丢到rowDataPipeDelegate,这里我们通过keys拿到selectTask中得到的数据
                            DbBatch dbBatch = rowDataPipeDelegate.get(keys);
                           ...
                            otterExtractorFactory.extract(dbBatch);// 重新装配一下数据
                            ...

                            List<PipeKey> pipeKeys = rowDataPipeDelegate.put(dbBatch, nextNodeId);
                            etlEventData.setDesc(pipeKeys);
                            ...
                        } 
                    }
                };

                // 构造pending任务，可在关闭线程时退出任务
                SetlFuture extractFuture = new SetlFuture(StageType.EXTRACT, etlEventData.getProcessId(),
                                                          pendingFuture, task);
                executorService.execute(extractFuture);
            } ...
        }
    }
```

#### 3.1 otterExtractorFactory.extract(dbBatch)

从otterExtractorFactory这个工厂类选择extractor进行处理。这里我们重点关注DatabaseExtractor。

`DatabaseExtractor`

```java
@Override
    public void extract(DbBatch dbBatch) throws ExtractException {
        ...
        // 读取配置
        Pipeline pipeline = getPipeline(dbBatch.getRowBatch().getIdentity().getPipelineId());
        boolean mustDb = pipeline.getParameters().getSyncConsistency().isMedia();
        boolean isRow = pipeline.getParameters().getSyncMode().isRow();// 如果是行记录是必须进行数据库反查
        // 读取一次配置
        adjustPoolSize(pipeline.getParameters().getExtractPoolSize()); // 调整下线程池，Extractor会被池化处理
        ExecutorCompletionService completionService = new ExecutorCompletionService(executor);

        // 进行并发提交
        ExtractException exception = null;
        // 每个表进行处理
        List<DataItem> items = new ArrayList<DataItem>();
        List<Future> futures = new ArrayList<Future>();
        List<EventData> eventDatas = dbBatch.getRowBatch().getDatas();
        for (EventData eventData : eventDatas) {
            //DDL语句忽略
            if (eventData.getEventType().isDdl()) {
                continue;
            }

            DataItem item = new DataItem(eventData);
            // 针对row模式，需要去检查一下当前是否已经包含row记录的所有字段，如果发现字段不足，则执行一次数据库查询
            boolean flag = mustDb
                           || (eventData.getSyncConsistency() != null && eventData.getSyncConsistency().isMedia());
            ...
            if (flag && (eventData.getEventType().isInsert() || eventData.getEventType().isUpdate())) {// 判断是否需要反查
                Future future = completionService.submit(new DatabaseExtractWorker(pipeline, item), null); // 提交进行并行查询
                ...
                futures.add(future);// 记录一下添加的任务
            }

            items.add(item);// 按顺序添加
        }

        // 开始处理结果
        int index = 0;
        while (index < futures.size()) { // 循环处理发出去的所有任务
            try {
                Future future = completionService.take();// 它也可能被打断
                future.get();
            } ...

            index++;
        }

        if (index < futures.size()) {
            // 小于代表有错误，需要对未完成的记录进行cancel操作，对已完成的结果进行收集，做重复录入过滤记录
            cancel(futures);
            throw exception;
        } else {
            // 全部成功分支, 构造返回结果也要保证原始的顺序
            for (int i = 0; i < items.size(); i++) {
                DataItem item = items.get(i);
                if (item.filter) { // 忽略需要被过滤的数据，比如数据库反查时记录已经不存在
                    eventDatas.remove(item.getEventData());
                }
            }
        }

    }
```

#### 3.2 数据并行查询

Future future = completionService.submit(new DatabaseExtractWorker(pipeline, item), null); // 提交进行并行查询

`DatabaseExtractor.rum()`

```java
        public void run() {
            try {
               ...
                // 获取数据表信息
                DataMedia dataMedia = ConfigHelper.findDataMedia(pipeline, eventData.getTableId());
                DbDialect dbDialect = dbDialectFactory.getDbDialect(pipeline.getId(),
                    (DbMediaSource) dataMedia.getSource());
                Table table = dbDialect.findTable(eventData.getSchemaName(), eventData.getTableName());
                TableData keyTableData = buildTableData(table, eventData.getKeys());
                ...

                boolean needAll = pipeline.getParameters().getSyncMode().isRow()
                                  || (eventData.getSyncMode() != null && eventData.getSyncMode().isRow());

                ...

                List<DataMediaPair> mediaParis = ConfigHelper.findDataMediaPairByMediaId(pipeline, dataMedia.getId());
                List<String> viewColumnNames = buildMaxColumnsFromColumnPairs(mediaParis, eventData.getKeys());
                // modified by ljh at 2012-11-04
                // 反查数据时只反查带update=true标识的数据，因为update=false的记录可能只是进行filter需要用到的数据，不需要反查
                TableData columnTableData = buildTableData(table,
                    eventData.getUpdatedColumns(),
                    needAll,
                    viewColumnNames);

                if (columnTableData.columnNames.length == 0) {
                    // 全主键，不需要进行反查
                } else {
                    //这里会根据主键到数据库查询数据
                    List<String> newColumnValues = select(dbDialect,
                        eventData.getSchemaName(),
                        eventData.getTableName(),
                        keyTableData,
                        columnTableData);

                    if (newColumnValues == null) {
                        ...
                    } else {
                        // 构造反查的返回结果
                        List<EventColumn> newEventColumns = new ArrayList<EventColumn>();
                        for (int i = 0; i < newColumnValues.size(); i++) {
                            EventColumn column = new EventColumn();
                            column.setIndex(columnTableData.indexs[i]);
                            column.setColumnName(columnTableData.columnNames[i]);
                            column.setColumnType(columnTableData.columnTypes[i]);
                            column.setNull(newColumnValues.get(i) == null);
                            column.setColumnValue(newColumnValues.get(i));
                            column.setUpdate(true);
                            newEventColumns.add(column);
                        }

                        // 处理下columns中不在反查字段内的字段列表
                        for (EventColumn column : eventData.getColumns()) {
                            boolean override = false;
                            for (EventColumn newEventColumn : newEventColumns) {
                                if (StringUtils.equalsIgnoreCase(newEventColumn.getColumnName(), column.getColumnName())) {
                                    override = true;
                                    break;
                                }
                            }

                            if (!override) {// 针对newcolumns不存在的记录进行添加
                                newEventColumns.add(column);
                            }
                        }

                        Collections.sort(newEventColumns, new EventColumnIndexComparable()); // 重新排个序
                        eventData.setColumns(newEventColumns);
                    }
                }
            } ...
        }
```

### 问题

1. extract的核心功能？
2. 反查数据库的作用在哪里？为什么要这么做？

### 参考文献

[Canal和Otter讨论二(原理与实践)]( https://www.cnblogs.com/yanshaoshuai/p/11987281.html )



