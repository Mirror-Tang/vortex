# vortex

Modular ZK proof layer

---

我们基于分布式计算资源可以为用户提供快速、安全、高效的零知识证明生成服务，即Prove服务。用户仅需通过我们提供的API接口，上传必需的文件（input_and_elf.json）与数据至我们的Prove端。依托于我们强大的后端GPU计算能力，能够在极短的时间内，准确无误地为用户生成所需的零知识证明。

Vortex是一个去中心化的ZK证明生成网络，它可以帮助用户生成ZK-SNARK证明并提交给智能合约验证。

Vortex智能合约是一套Solidity代码书写的可升级智能合约，可以部署在不同的区块链上，它会内嵌API以连接Vortex网络。  

Vortex API既可以在智能合约中调用，也可以在项目的web2代码部分直接使用，通过Vortex API 用户先向Vortex Hub声明证明任务，然后由Vortex Hub将对证明任务进行分配。  

Vortex Hub负责证明任务广播，它将任务广播到Vortex网络中，一定时间内响应的证明者会随机获取证明生成任务。Vortex Hub也负责用户证明请求承接，用户在发送需要生成证明的具体内容和电路前需要先向Vortex Hub声明任务，然后Vortex Hub会在Vortex网络中广播该任务。 

#  Vortex GPU机密计算技术介绍

## 机密GPU的机密计算⼯作模式

![image](https://github.com/monkAlmond/vortex/blob/master/image/1.png)

• CC（Confidential Computing）⼯作模式必须在GPU启动前提前配置好。在云场景中由VMM来设置；将CC⼯作模式位配置到GPU EEPROM中，然后执⾏ 
  GPU reset才能⽣效。
• GPU attestation report会记录GPU⼯作在哪⼀种CC⼯作模式下。确保不可信的hypervisor只能按照预期来设置CC⼯作模式。
• GPU reset会清空GPU内存和状态，包括会话密钥和secrets。

## CC工作模式开启后如何保护内存？

![image](https://github.com/monkAlmond/vortex/blob/master/image/2.png)

• 大多数GPU内存被配置到Compute Protected Region（CPR），该区域的内存由GPU内部的硬件firewalls提供保护；
• 少部分GPU内存无需保护：
	用于保存加密的CUDA命令buffer
	用于NVLINK端到端通信的加密bounce buffer
• CVM中的NV驱动命令GPU在未保护内存区域中分配bounce buffer，并映射到CPU的共享内存区域中;
• 后续NV驱动会使用SPDM会话密钥将加密数据写入到bounce buffer中。

## 机密计算如何保护CUDA程序（cuda程序无需改变）?

![image](https://github.com/monkAlmond/vortex/blob/master/image/3.png)

• CPU-GPU间的全部通信全都是加密的，包括数据传输、命令和CUDA kernels。
•由于IOMMU禁⽌设备直接访问CVM私有内存，因此CPU-GPU之间需要使⽤位于共享内存中的bounce buffer进行数据交换。
• 向GPU设备发送数据时：
	位于CVM内的NVIDIA驱动将待发送的数据进行加密，再写⼊到bounce buffer中；
	GPU DMA引擎从bounce buffer中读取加密数据，再解密到GPU的受保护内存中。
• 从GPU设备读取数据时：
	GPU DMA引擎通过PCIe总线将经过加密的数据写入到bounce buffer中；
	位于CVM内的NVIDIA驱动从bounce buffer读取加密数据，再解密到CVM私有内存中。

## 机密GPU的内部保护机制

![image](https://github.com/monkAlmond/vortex/blob/master/image/4.png)

如果GPU在启动时开启了CC工作模式，会阻断所有对GPU CPR内存的入站和出站访问。
• PCIe Firewall负责阻断：
	来⾃CPU侧对绝大多数GPU寄存器的访问
	对GPU CPR内存的任何访问
• NVLink Firewall负责阻断来自peer GPU对GPU CPR内存的任何访问。 
• DMA引擎只能读取或写入GPU CPR（计算保护单元）之外的内存。 
• 阻断所有其他引擎（比如计算SMs）访问CPR之 外的内存。 
开启CC⼯作模式后，会禁⽤所有的GPU性 能计数器，免受侧信道攻击。

## 基于CVM的多GPU机密计算

Confidential Virtual Machine.

![image](https://github.com/monkAlmond/vortex/blob/master/image/5.png)

• GPU之间使⽤NVLink进⾏设备间通信。 
	开启CC⼯作模式后不⽀持PCIe的P2P；
	CUDA APIs和硬件firewall都禁⽌对peer GPU内存直接进行指针解引用。
• 源GPU和目的GPU侧的DMA引擎使用共享的会话密钥来保护NVLink上的传输。 cudaMemcpyDeviceToDevice()通过bounce buffer传输加密数据。
	源GPU侧的DMA引擎负责加密数据，再通过不可信的 NVLink将加密数据传输到⽬的GPU侧的未经保护内存中； 
	目的GPU侧的DMA引擎负责将bounce buffer中的加密数据 解密到自己的GPU保护内存中。

## 机密MIG

![image](https://github.com/monkAlmond/vortex/blob/master/image/6.png)

相关技术：
• NVIDIA vGPU：允许多个VM实例共享⼀个NVIDIA GPU PF。
• Multi-Instance GPU（MIG）：将GPU资源划分为多个GPU实例，可视为多个⼦GPU
• SR-IOV：将GPU VF暴露为PCIe设备并透传给VM。

## GPU TEE依赖的安全特性

Trusted execution environment.

• 片上的RoT：RoT负责在设备启动和运行时确保软件和硬件的完整性
• NVIDIA RISC-V微控制器：处理特定的安全任务和操作，如密钥管理和安全启动
• 加密的固件：固件在存储或传输时使用加密保护，以防止泄露或未授权修改
• FIPS 140-3 Level 2 密码加密：用于防止未授权访问加密密钥
• 设备证书：证明设备身份的数字证书，用于设备在网络上进行身份验证和安全通信
• Secure Boot：确保设备只加载和执行经过验证的，可信的操作系统和软件
• Measured Boot：是一种保证启动过程安全的机制，通过度量（记录和验证）启动过程中加载的每个组件的完整性，确保设备从可信的基线状态启动
• Hardware Fault Injetion Counter：用于抵抗通过敌意引起硬件错误（如电压、温度变化）来篡改或绕过安全机制的攻击
• 固件Revocation：用于废除或撤销过时或已知存在安全漏洞的固件版本以保证设备安全
• SR-IOV for Secure MIG：用于单GPU多VM，确保每个虚拟GPU实例在资源使用上的隔离和安全
• PKC固件认证：使用公钥加密技术来验证固件的完整性和来源，确保固件未被篡改
• Unique Identity Key Pair：指设备具有一对独特的公钥和私钥，用于各种安全功能，如加密通信、数字签名等




# RISC ZERO测试内容





## Prove流程概述

1. 用户向ProveAPI发起包含`input_and_elf.json`的文件内容的POST请求，这个请求会初始化一个新的Prove任务。作为响应，API将返回给用户一个`task_id`，作为后续跟踪任务进度和执行状态的依据。
2. API接收`input`以及`elf`并传送至Prove端。
3. Prove端读取`elf`和`input`，并利用GPU加速生成证明（即`receipt.json`文件）。`receipt.json`文件会返回给任务调度系统。任务调度系统将`receipt.json`文件和相应的执行结果与之前生成的`task_id`相关联，将`task_id`、`receipt`、`elf`和`input`进行持久化存储，以供后续查询。
4. Prove端会通过API返回`receipt`给用户。用户可根据`receipt`中的信息，来验证证明的有效性或执行其他相关操作。

## 生成配置文件input_and_elf.json

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

3. 相应的，用户应在与build.rs同目录的Cargo.toml中添加risc0-zkvm和serde的依赖。其他的依赖取决于用户具体希望构建的input，用户需自行补充。

4. 接下来，用户在/methods目录执行cargo build，该命令能够在项目根目录中/target路径下生成input_and_elf.json文件，其中包含电路的input以及elf编码。

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

## 创建新的Prove任务

### 概述

此接口用于创建一个新的Prove任务。用户通过发送一个POST请求到`api/v1/prove/create`来创建任务。成功创建任务后，接口将返回一个`task_id`，用户可使用此`task_id`来查询或管理Prove任务。

### 请求

- **方法:** POST
- **URL:** `/api/v1/prove/create`
- **Content-Type:** `application/json`
  
#### 请求参数

通过POST body的形式提供：

- **input** - 类型: `str` - 输入给电路的input
- **elf** - 类型: `str` - 电路编译后产生的二进制文件，被编码为u8形式

### 响应

#### 响应体

响应内容为JSON格式，包含以下字段：

- **code:** 状态码
- **msg:** 信息
- **results:** 新创建的Prove任务的唯一标识符。

#### 成功响应示例

```json
{
 "code": 0,
 "msg": "successfully",
 "results": {}
}
```

#### 响应状态码

- **200 OK:** 请求成功，Prove任务已被创建。
- **400 Bad Request:** 请求的格式错误或缺少必要信息。
- **401 Unauthorized:** 用户未认证，请求被拒绝。
- **500 Internal Server Error:** 服务器内部错误，无法完成请求。

## 查询Prove任务执行状态

### 概述

此接口允许用户查询指定Prove任务的当前执行状态。通过发送一个GET请求到`api/v1/prove/query`并附上任务的`task_id`，用户可以获取任务的状态信息。

### 请求

- **方法:** GET
- **URL:** `/api/v1/prove/query`
- **Content-Type:** `application/json`
  
#### 请求参数

通过URL参数的形式提供：

- **task_id** - 类型: `str` - 要查询状态的Prove任务的唯一标识符。

### 响应

#### 响应体

响应内容为JSON格式，包含以下字段：

- **code:** 状态码
- **msg:** 信息
- **results:** Receipt

#### 成功响应示例

```json
{
 "code": 0,
 "msg": "successfully",
 "results": {}
}
```

#### 响应状态码

- **200 OK:** 请求成功，成功返回任务状态。
- **400 Bad Request:** 请求的格式错误或缺少必要信息。
- **401 Unauthorized:** 用户未认证，请求被拒绝。
- **500 Internal Server Error:** 服务器内部错误，无法完成请求。

---

# Circom测试内容


# Halo2测试内容


