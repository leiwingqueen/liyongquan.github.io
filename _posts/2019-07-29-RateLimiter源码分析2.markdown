---
layout: post
title:  "RateLimiter-Դ�����2"
description: RateLimiter-Դ�����2
date:   2019.07.29 22:37:56 +0530
categories: �ֲ�ʽ ���� RateLimiter
---
####  һ��ǰ��
��һƪ��β��ʱ���������������⣬��һ����Ҫ����������������˼�������ۡ�
![ticket����](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/20190915094151.png)

- ����1
�����returnValueΪʲô����ֱ�ӷ���nextFreeTicketMicros������ֱ�ӻ�ȡnextFreeTicketMicros����ǰ��ֵ��
- ����2
���timeout����Ϊ0�����Ǵ�����ticket���������£���ҪfreshPermits����䣬��ô���ղ���Ҳ��sleep��
#### �����������
![������־](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/20190915095032.png)
�ҳ�����sleep֮ǰ����һ����־�����ڹ۲����˯�ߵ�ʱ�䡣
�򵥱�дһ�����������ڲ���ticket����ĳ�����

```
public static void main(String[] args) {
    RateLimiter rateLimiter=RateLimiter.create(5);
    for(int i=0;i<5;i++) {
      boolean acquire = rateLimiter.tryAcquire(10);
      System.out.println("result:"+acquire);
    }
  }
```
�������еĽ����
```
sleep0ms...
result:true
result:false
result:false
result:false
result:false

Process finished with exit code 0
```
���ǿ���֪��
- ���п���ticket������ticket�����ʱ��᷵�سɹ������Ҳ���sleep
- �����ticket�������һ�ε�nextFreeTicketMicros��������һ��tryAcquireʧ�ܡ������������ʾ��nextFreeTicketMicros���޸�Ϊ2s��(10/5=2)��

���������ܽ��£�ֻҪ����ͷ��������һ��ticket(��ǰʱ��>=nextFreeTicketMicros)����������������Ҫ���ٸ�ticket�����Ƕ�һ���������������true�����Ұ�ticket����Ϊ����(�������nextFreeTicketMicros)����������ģʽ���������˾������ˣ�����ǰ��һ���û������˺ܶ�ticket����ô��������Ҫ�ȴ��ܳ���ʱ��������뵽һ��ticket��
#### �ġ��ܽ�
RateLimiter��SmoothBurstyʵ�������Ѿ������������ˣ����Կ��ó�����ʵSmoothBursty��ʵ�ֻ����һЩ�׶ˣ�����˵���ܻ���ֱȽ����ص�ͻ������������һ��ʵ����SmoothWarmingUp�Ƿ��ܱȽϺõؽ���ⲿ�����⣿����д�����







