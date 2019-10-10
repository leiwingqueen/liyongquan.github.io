---
layout: post
title:  "Disruptor-源码分析(二)"
description: Disruptor-源码分析(二)
date:   2019-10-10 14:52:00 +000
categories: mq 高并发 disruptor
tags: mq 高并发 disruptor
---

在分析Disruptor的消费者的处理逻辑的时候，如果有多个消费者，Disruptor如何处理这些消费者的竞争关系？
handleEventsWith和handleEventsWithWorkerPool又有什么区别？

我自己简单写了一个测试类来进行测试。
```java
package com.lmax.disruptor.example;

import com.lmax.disruptor.*;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;
import com.lmax.disruptor.util.DaemonThreadFactory;

/**
 * @Auther: 李泳权
 * @Date: 2019/10/10 14:05
 * @Description:
 */
public class SimpleDisruptorTest {
    static final int RING_SIZE = 8;
    public static void main(String[] args) throws InterruptedException
    {
        Disruptor<SimpleMessage> disruptor = new Disruptor<>(
                SimpleMessage.FACTORY, RING_SIZE, DaemonThreadFactory.INSTANCE, ProducerType.MULTI,
                new BlockingWaitStrategy());
        //disruptor.handleEventsWith(new Consumer()).then(new Consumer());
        //disruptor.handleEventsWith(new Consumer[]{new Consumer(),new Consumer()});
        disruptor.handleEventsWithWorkerPool(new Consumer[]{new Consumer(),new Consumer()});
        final RingBuffer<SimpleMessage> ringBuffer = disruptor.getRingBuffer();
        Publisher p = new Publisher();
        System.out.println("publishing " + RING_SIZE + " messages");
        int i = 0;
        for (; i < RING_SIZE; i++)
        {
            ringBuffer.publishEvent(p, (long)i);
            Thread.sleep(10);
        }
        System.out.println("start disruptor");
        disruptor.start();
        System.out.println("continue publishing");
        while (true)
        {
            ringBuffer.publishEvent(p, (long)i);
            Thread.sleep(10);
            i++;
        }
    }

    public static class Publisher implements EventTranslatorOneArg<SimpleMessage, Long>
    {
        @Override
        public void translateTo(SimpleMessage message, long sequence,Long value)
        {
            message.setValue(value);
            message.setSequence(sequence);
        }
    }

    public static class Consumer implements EventHandler<SimpleMessage>,WorkHandler<SimpleMessage>
    {
        @Override
        public void onEvent(SimpleMessage event, long sequence, boolean endOfBatch) throws Exception
        {
            System.out.println("sequence:"+sequence+","+"event:"+event.toString());
        }

        @Override
        public void onEvent(SimpleMessage event) throws Exception {
            System.out.println("event:"+event.toString());
        }
    }

    public static class SimpleMessage{
        private long value;
        private long sequence;
        private static final EventFactory<SimpleMessage> FACTORY = new EventFactory<SimpleMessage>()
        {
            @Override
            public SimpleMessage newInstance()
            {
                return new SimpleMessage();
            }
        };

        public long getValue() {
            return value;
        }

        public void setValue(long value) {
            this.value = value;
        }

        public long getSequence() {
            return sequence;
        }

        public void setSequence(long sequence) {
            this.sequence = sequence;
        }

        @Override
        public String toString() {
            return "SimpleMessage{" +
                    "value=" + value +
                    ", sequence=" + sequence +
                    '}';
        }
    }
}

```

