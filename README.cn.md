# vortex

Modular ZK proof layer

---

# Prove流程概述

1. 用户向ProveAPI发起包含`input_and_elf.json`的文件内容的POST请求，这个请求会初始化一个新的Prove任务。作为响应，API将返回给用户一个`task_id`，作为后续跟踪任务进度和执行状态的依据。
2. API接收`input`以及`elf`并传送至Prove端。
3. Prove端读取`elf`和`input`，并利用GPU加速生成证明（即`receipt.json`文件）。`receipt.json`文件会返回给任务调度系统。任务调度系统将`receipt.json`文件和相应的执行结果与之前生成的`task_id`相关联，将`task_id`、`receipt`、`elf`和`input`进行持久化存储，以供后续查询。
4. Prove端会通过API返回`receipt`给用户。用户可根据`receipt`中的信息，来验证证明的有效性或执行其他相关操作。

# 创建新的Prove任务

## 概述

此接口用于创建一个新的Prove任务。用户通过发送一个POST请求到`api/v1/prove/create`来创建任务。成功创建任务后，接口将返回一个`task_id`，用户可使用此`task_id`来查询或管理Prove任务。

## 请求

- **方法:** POST
- **URL:** `/api/v1/prove/create`
- **Content-Type:** `application/json`
  
### 请求参数

通过POST body的形式提供：

- **input** - 类型: `str` - 输入给电路的input
- **elf** - 类型: `str` - 电路编译后产生的二进制文件，被编码为u8形式

## 响应

### 响应体

响应内容为JSON格式，包含以下字段：

- **code:** 状态码
- **msg:** 信息
- **results:** 新创建的Prove任务的唯一标识符。

### 成功响应示例

```json
{
 "code": 0,
 "msg": "successfully",
 "results": {}
}
```

### 响应状态码

- **200 OK:** 请求成功，Prove任务已被创建。
- **400 Bad Request:** 请求的格式错误或缺少必要信息。
- **401 Unauthorized:** 用户未认证，请求被拒绝。
- **500 Internal Server Error:** 服务器内部错误，无法完成请求。

# 查询Prove任务执行状态

## 概述

此接口允许用户查询指定Prove任务的当前执行状态。通过发送一个GET请求到`api/v1/prove/query`并附上任务的`task_id`，用户可以获取任务的状态信息。

## 请求

- **方法:** GET
- **URL:** `/api/v1/prove/query`
- **Content-Type:** `application/json`
  
### 请求参数

通过URL参数的形式提供：

- **task_id** - 类型: `str` - 要查询状态的Prove任务的唯一标识符。

## 响应

### 响应体

响应内容为JSON格式，包含以下字段：

- **code:** 状态码
- **msg:** 信息
- **results:** Receipt

### 成功响应示例

```json
{
 "code": 0,
 "msg": "successfully",
 "results": {}
}
```

### 响应状态码

- **200 OK:** 请求成功，成功返回任务状态。
- **400 Bad Request:** 请求的格式错误或缺少必要信息。
- **401 Unauthorized:** 用户未认证，请求被拒绝。
- **500 Internal Server Error:** 服务器内部错误，无法完成请求。

---
