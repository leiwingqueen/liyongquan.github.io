---
layout: post
title:  "Disruptor-源码分析(二)"
description: Disruptor-源码分析(二)
date:   2019-10-10 14:52:00 +000
categories: mq 高并发 disruptor
tags: mq 高并发 disruptor
---

这一篇重点看下消费者的处理逻辑，我们来分析下disruptor为什么快？

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

对于同一条消息m，handleEventsWith会让每个消费者都分别消费一遍(消息广播)。而handleEventsWithWorkerPool的多个消费者会竞争消费，对于一条消息只会被一个消费者消费。

我们看disruptor的方法介绍

```java
    /**
     * Set up a {@link WorkerPool} to distribute an event to one of a pool of work handler threads.
     * Each event will only be processed by one of the work handlers.
     * The Disruptor will automatically start this processors when {@link #start()} is called.
     *
     * @param workHandlers the work handlers that will process events.
     * @return a {@link EventHandlerGroup} that can be used to chain dependencies.
     */
    @SafeVarargs
    @SuppressWarnings("varargs")
    public final EventHandlerGroup<T> handleEventsWithWorkerPool(final WorkHandler<T>... workHandlers)
    {
        return createWorkerPool(new Sequence[0], workHandlers);
    }
```



- handleEventsWith

```mermaid
graph LR
m1-->c1(consumer1)
m1-->c2(consumer2)
m2-->c1(consumer1)
m2-->c2(consumer2)
```

- handleEventsWithWorkerPool

```mermaid
graph LR
m1-->c1(consumer1)
m2-->c2(consumer2)
```



我们关注下下面的两个类WorkProcessor和BatchEventProcessor

```mermaid
graph BT
E(EventProcessor)
WorkProcessor-->E
BatchEventProcessor-->E
```

handleEventsWith会创建BatchEventProcessor对象，handleEventsWithWorkerPool会构建WorkProcessor。

我们看下官方的类描述

```java
/**
 * Convenience class for handling the batching semantics of consuming entries from a {@link RingBuffer}
 * and delegating the available events to an {@link EventHandler}.
 * <p>
 * If the {@link EventHandler} also implements {@link LifecycleAware} it will be notified just after the thread
 * is started and just before the thread is shutdown.
 *
 * @param <T> event implementation storing the data for sharing during exchange or parallel coordination of an event.
 */
public final class BatchEventProcessor<T>
    implements EventProcessor
```

BatchEventProcessor是批处理事件模型。

```java
/**
 * <p>A {@link WorkProcessor} wraps a single {@link WorkHandler}, effectively consuming the sequence
 * and ensuring appropriate barriers.</p>
 *
 * <p>Generally, this will be used as part of a {@link WorkerPool}.</p>
 *
 * @param <T> event implementation storing the details for the work to processed.
 */
public final class WorkProcessor<T>
    implements EventProcessor
{
```

WorkProcessor对于每条消息只会处理一次。

我们看下BatchEventProcessor的处理逻辑。

```java
    private void processEvents()
    {
        T event = null;
        long nextSequence = sequence.get() + 1L;

        while (true)
        {
            try
            {
                final long availableSequence = sequenceBarrier.waitFor(nextSequence);
                if (batchStartAware != null)
                {
                    batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
                }

                while (nextSequence <= availableSequence)
                {
                    event = dataProvider.get(nextSequence);
                    //事件触发
                    eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
                    nextSequence++;
                }

                sequence.set(availableSequence);
            }
            ...
        }
    }

```

一开始nextSequence = sequence.get() + 1L;

availableSequence相当于写入的尾指针

```mermaid
graph LR
m0("-1")-->m1(0)
m1-->m2(1)
m2-->m3(2)
m3-->m4(3)
m4-->m5(4)
subgraph availableSequence
m4
end
subgraph sequence
m0
end
subgraph nextSequence
m1
end

```

nextSequence会不断地往后移。当nextSequence>availableSequence，会把sequence更新为availableSequence。然后再读取下一个availableSequence的位点

```mermaid
graph LR
m0("-1")-->m1(0)
m1-->m2(1)
m2-->m3(2)
m3-->m4(3)
m4-->m5(4)
subgraph availableSequence
m4
end
subgraph sequence
m0
end
subgraph nextSequence
m2
end
```



```mermaid
graph LR
m0("-1")-->m1(0)
m1-->m2(1)
m2-->m3(2)
m3-->m4(3)
m4-->m5(4)
subgraph sequence
subgraph availableSequence
m4
end
end
subgraph nextSequence
m5
end
```

由于BatchEventProcessor是并行的模型，对于sequence和nextSequence并不会产生竞争，因此也需要锁来协调。

我们需要重点关注的是WorkProcessor这个类。

```java
            try
            {
                // if previous sequence was processed - fetch the next sequence and set
                // that we have successfully processed the previous sequence
                // typically, this will be true
                // this prevents the sequence getting too far forward if an exception
                // is thrown from the WorkHandler
                if (processedSequence)
                {
                    processedSequence = false;
                    do
                    {
                        nextSequence = workSequence.get() + 1L;
                        sequence.set(nextSequence - 1L);
                    }
                    while (!workSequence.compareAndSet(nextSequence - 1L, nextSequence));
                }

                if (cachedAvailableSequence >= nextSequence)
                {
                    event = ringBuffer.get(nextSequence);
                    workHandler.onEvent(event);
                    processedSequence = true;
                }
                else
                {
                    cachedAvailableSequence = sequenceBarrier.waitFor(nextSequence);
                }
            }
```

我们用一个图来模拟整个过程

```mermaid
graph LR
subgraph 消费者指针
m2(1)
end
m1(0)-->m2
m2-->m3(2)
subgraph 生产者指针
m4(3)
end
m3-->m4
m4-->m5(4)
c1(消费者1)
c2(消费者2)
c1==>|抢消息1|m2
c2==>|抢消息1|m2
```

假设消费者1抢成功，并成功把 消费者的指针往下挪一位。

```mermaid
graph LR
subgraph 消费者指针
m3
end
m1(0)-->m2(1)
m2-->m3(2)
subgraph 生产者指针
m4(3)
end
m3-->m4
m4-->m5(4)
c1(消费者1)
c2(消费者2)
c1==>|抢成功|m2
c1==>|事件处理|c1
c2==>|继续抢消费者指针|m3
```

消费者1抢到消息1，然后接下来进行事件处理。抢不到消费的消费者2继续抢消费者指针。

```mermaid
graph LR
m1(0)-->m2(1)
m2-->m3(2)
subgraph 生产者指针
subgraph 消费者指针
m4(3)
end
end
m3-->m4
m4-->m5(4)
c1(消费者1)
c2(消费者2)
c1==>|继续抢消费者指针|m4
c2==>|抢成功|m3
c2==>|事件处理|c2
```

消费者2抢消息2成功，接着继续处理消息。消费者1抢消息3

```mermaid
graph LR
m1(0)-->m2(1)
m2-->m3(2)
subgraph 生产者指针
m4(3)
end
subgraph 消费者指针
m5
end
m3-->m4
m4-->m5(4)
c1(消费者1)
c2(消费者2)
c1==>|抢成功|m4
c1==>|事件处理|c1
c2==>|等待新消息|m5
```

### 总结

- 在disruptor的广播模式下，消费者不需要对锁进行竞争，因此处理的性能取决于消费者的处理速度
- 在disruptor的点对点模式下，消费者需要对消息资源进行竞争，竞争是通过CAS来完成。抢到消息后会马上把指针移到下一个位置(更严谨的说法应该是能把指针成功往后移动一位才算抢成功)，然后再进行事件处理。这样处理的优点是资源竞争的点只有在移动指针的那一刻，事件处理的过程是不需要产生资源竞争。有点类似大家排队去饭堂吃饭，在排队窗口大家会竞争，但是拿到饭菜后不需要堵在窗口，回到餐桌上慢慢吃。吃完后再去排队。

### 问题

- 消费者的处理逻辑个人认为并没有很出彩的逻辑，那么回到最早的问题？disruptor为什么快？

