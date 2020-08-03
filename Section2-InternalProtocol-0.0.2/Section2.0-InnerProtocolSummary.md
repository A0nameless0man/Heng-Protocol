# 内部通信协议 0.0.2

## 综述

### 协议目的

1. 协调评测机和控制端的通信。

2. 验证评测机的身份，防止恶意攻击

3. 兼容缓存机制的运行

## 名称解释

### 评测机端的SecrectKey

评测机端的 SecrectKey 。由控制端生成，与 ackey 相关联。由管理员填入评测机的配置文件中。如不慎泄密或失密，应使用控制端的管理功能注销对应的密钥对。

### 评测机端的ackey

用于识别不同的评测机及其密钥，无保密必要

### 任务令牌

流量控制系统的组成部分。令牌与评测机关联。向评测机下发任务需占用一个对应评测机的任务令牌并登记到活动任务令牌队列中。当评测结束并返回后，释放任务令牌至可用池中。

## 组成部分

消息的基本结构如 `BasicMessage` 所示，含有 `contextID` , `type` 和 `body` 三个字段。

- `contextID` 是一个指明消息的订阅方的字符串

- `type` 指明了消息的类型

- `body` 包含了根据类型不同的具体内容

对于没有收到下一步的回应的情况，采用指数退避算法至多重发 5 次

对于只需简单返回 ack 的情况，即使重复收到消息，亦应为每次收到消息单独返回 ack

### 评测准备协议

此协议应当在连接后立即进行，具有认证合法评测端的作用。

其中，签名按照下述方法进行

- 将 `signature` 字段填写为 `7def6260-cf55-11ea-87d0-0242ac130003`

- 对 `JudgerInfo` 使用 `JSON.stringify` 函数处理为字符串。

- 使用如下函数得到签名

```javascript
let signature = crypto.createHash('sha256')
      .update(JudgerInfoString)
      .update(SecrectKey).digest('hex')
```

#### 评测准备协议前提

##### 评测准备协议评测机端

- 已经通过身份认证拿到了会话令牌和 `JudgerID`

- 配置文件写入评测机端的 SecrectKey 和 ackey

- 配置文件写入评测机地址

##### 评测准备协议控制端

- 记录发放的评测机的 SecrectKey 和 ackey

#### 评测准备协议内容

- 评测机通过  `WebSocket` 链接向控制端发送如下内容

  `评测机自述消息`
  `JudgerInfo`

- 控制端发送确认信号（`AckMessage`）

  - 如未收到该确认信号，评测机应重试，失败 5 次后断开连接

### 心跳协议

#### 心跳协议前提

##### 心跳协议评测机端

1. 已经通过身份认证拿到了会话令牌和 `JudgerID`

##### 心跳协议控制端

1. 已经通过身份认证发放了会话令牌

#### 心跳协议内容

- 评测机定期通过  `WebSocket` 链接向控制端发送如下内容

  `评测机心跳消息`
  `StatusReportMessage`

- 控制端发送确认信号（`AckMessage`）

  - 如未收到该确认信号，评测机应重试，失败 5 次后断开连接

#### 心跳协商

- 控制端发送 `StatusRequestPayload` 来设置心跳间隔，和/或要求立即报告。

- 评测端发送确认信号（`AckMessage`）

  - 子消息为 `StatusReportMessage`

### 评测任务协议

#### 评测任务协议前提

##### 评测任务协议评测机端

1. 已经通过身份认证拿到了会话令牌和 `JudgerID`

##### 评测任务协议控制端

1. 已经通过身份认证发放了会话令牌

2. 已经完成评测准备协议，拿到了任务令牌

#### 评测任务协议内容

##### 评测任务发放

- 控制端通过 `WebSocket` 链接向评测机发送如下内容，同时内部登记令牌为发送状态
  - 任务ID是控制端生成的一个ID对应一次发送尝试
    可能出现多个任务ID对应一个评测请求的情况（但有效ID只能有一个）

  - 每次发送任务都应当生成任务ID

  `任务字段类型`
  `JudgeRequest`

- 评测机返回确认（`Response`）

  - 如未收到回应，认为任务发送失败
    - 将任务放回控制端等待队列
    - 作废此次生成的任务ID
    - 释放对应令牌

- 控制端登记令牌为活动状态

  - 无需发送消息

- 当评测到达特定状态，评测端发送 `JudgeStateMessage`

- 当评测机评测结束，向控制端发送 `JudgeResult`

- 控制端接受结果并返回确认（`Response`）