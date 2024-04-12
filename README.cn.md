# vortex

Modular ZK proof layer

---

我们基于分布式计算资源可以为用户提供快速、安全、高效的零知识证明生成服务，即Prove服务。用户仅需通过我们提供的API接口，上传必需的文件（input_and_elf.json）与数据至我们的Prove端。依托于我们强大的后端GPU计算能力，能够在极短的时间内，准确无误地为用户生成所需的零知识证明。

# Prove流程概述

1. 用户向ProveAPI发起包含`input_and_elf.json`的文件内容的POST请求，这个请求会初始化一个新的Prove任务。作为响应，API将返回给用户一个`task_id`，作为后续跟踪任务进度和执行状态的依据。
2. API接收`input`以及`elf`并传送至Prove端。
3. Prove端读取`elf`和`input`，并利用GPU加速生成证明（即`receipt.json`文件）。`receipt.json`文件会返回给任务调度系统。任务调度系统将`receipt.json`文件和相应的执行结果与之前生成的`task_id`相关联，将`task_id`、`receipt`、`elf`和`input`进行持久化存储，以供后续查询。
4. Prove端会通过API返回`receipt`给用户。用户可根据`receipt`中的信息，来验证证明的有效性或执行其他相关操作。

# 生成配置文件input_and_elf.json

用户需要在本地生成包含input和elf的配置文件input_and_elf.json，具体操作步骤如下：

1. 用户需首先将新的lib.rs文件覆盖risc-build包中的lib.rs文件。

如果用户的当前项目是通过`cargo risczero new my_project  --guest-name my_guest`生成，则该项目中lib.rs文件路径位于~/.cargo/registry/src/your-rust-mirror/risc0-build-0.xx/src/lib.rs。

如果用户的当前项目clone于risc0，则该项目中lib.rs文件路径位于/risc0/build/src/lib.rs。

2. 覆盖后，用户还需将原本在/host中构建的电路input转移到/methods/src/build.rs中，并在build.rs中补充下列代码：

```
use risc0_zkvm::serde::Serializer;

pub fn to_vec<T>(value: &T) -> Result<Vec<u32>, risc0_zkvm::serde::Error>
where
    T: serde::Serialize + ?Sized,
{
    // Use the in-memory size of the value as a guess for the length
    // of the serialized value.
    let mut vec: Vec<u32> = Vec::with_capacity(core::mem::size_of_val(value));
    let mut serializer = Serializer::new(&mut vec);
    value.serialize(&mut serializer)?;
    Ok(vec)
}
```

下面是一个例子：

以变量a、b作为电路的两个input为例，它们均需经过to_vec()函数变成Vec<u32>形式，并封装为Vec<Vec<u32>>类型传入risc0_build::embed_methods()。在本例中，构造好的build.rs示例代码如下：

```
use risc0_zkvm::serde::Serializer;

pub fn to_vec<T>(value: &T) -> Result<Vec<u32>, risc0_zkvm::serde::Error>
where
    T: serde::Serialize + ?Sized,
{
    let mut vec: Vec<u32> = Vec::with_capacity(core::mem::size_of_val(value));
    let mut serializer = Serializer::new(&mut vec);
    value.serialize(&mut serializer)?;
    Ok(vec)
}

fn main() {
    let a: u64 = 17;
    let b: u64 = 23;
    let a_vec = to_vec(&a).unwrap();
    let b_vec = to_vec(&b).unwrap();

    let mut container = Vec::new();
    container.push(a_vec);
    container.push(b_vec);

    risc0_build::embed_methods(container);
}
```

相应的，用户应在与build.rs同目录的Cargo.toml中添加risc0-zkvm和serde的依赖。其他的依赖取决于用户具体希望构建的input，用户需自行补充。

接下来，用户在/methods目录执行cargo build，该命令能够在项目根目录中/target路径下生成input_and_elf.json文件，其中包含电路的input以及elf编码。

input_and_elf.json示例：

```
{
"input": [
[17, 0],
		[23, 0]
],
	"elf": [
127,
 …,
 0
]
}
```

用户在本地生成配置文件input_and_elf.json后，需要和我们的API交互。我们可以为用户创建一个新的Prove任务，用户也可以追踪自己的Prove任务的状态。

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
