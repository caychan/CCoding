- [大纲](#大纲)
  - [生产者](#生产者)
    - [消息体](#消息体)
    - [消息发送类型-普通消息、顺序消息](#消息发送类型-普通消息顺序消息)
    - [消息发送类型-延迟消息](#消息发送类型-延迟消息)
    - [消息发送类型-事务消息](#消息发送类型-事务消息)
  - [消费者](#消费者)
    - [消费方式](#消费方式)
    - [消费过滤](#消费过滤)
    - [消费重试](#消费重试)

> github求star：https://github.com/caychan/CCoding

最近看了下rocketmq官网，整理了一个简单的rocketmq相关知识点的xmind，主要是与生产者和消费者客户端相关。且不涉及底层原理和源码，但可以对rocketmq的使用有一个基本了解。

## 大纲
![20220929104109](https://caychan.oss-cn-beijing.aliyuncs.com/blog/87befe1db6a12d9442caa1f6ab4c68f5.png)


### 生产者
#### 消息体
![20220929104806](https://caychan.oss-cn-beijing.aliyuncs.com/blog/11c678b58ea3678128fe2056e56bf2bb.png)

#### 消息发送类型-普通消息、顺序消息
![20220929104945](https://caychan.oss-cn-beijing.aliyuncs.com/blog/a643bbe4747d00189c0870c0eca0b75a.png)

#### 消息发送类型-延迟消息
![20220929105028](https://caychan.oss-cn-beijing.aliyuncs.com/blog/c10aff240ad64ec564ddb7b7333bf13c.png)

#### 消息发送类型-事务消息
![20220929105102](https://caychan.oss-cn-beijing.aliyuncs.com/blog/8dc1dc7e3c595f4f31781a9b1f8d3d05.png)

其中的分布式事务模型的流程图来自官网：
![20220929105134](https://caychan.oss-cn-beijing.aliyuncs.com/blog/0eb3dac4319103e8e6f5035b4457c312.png)



### 消费者
#### 消费方式
![20220929105340](https://caychan.oss-cn-beijing.aliyuncs.com/blog/d3dbc0de96549c5214aaaf16a17d36ba.png)

#### 消费过滤
![20220929105601](https://caychan.oss-cn-beijing.aliyuncs.com/blog/c4f6b8c0451d5cecd93f32077eeb01f2.png)

#### 消费重试
![20220929105618](https://caychan.oss-cn-beijing.aliyuncs.com/blog/aa6b7eac85e137e984c87cf1869a99b4.png)



> rocketmq官网文档地址：https://rocketmq.apache.org/docs/4.x/


