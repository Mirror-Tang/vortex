# Vortex
Vortex is a decentralized ZK proof generation network that helps users generate ZK-SNARK proofs and submit them for verification by smart contracts.

![image](https://github.com/Mirror-Tang/vortex/blob/master/vortexlogo.png)

## Vortex Architecture

### Vortex Smart Contract

The Vortex smart contract is a set of upgradable smart contracts written in Solidity that can be deployed on various blockchains. It includes embedded APIs to connect to the Vortex network.

### Vortex API

The Vortex API can be invoked within smart contracts as well as directly used in the web2 code section of a project. Users initially declare proof tasks to the Vortex Hub through the Vortex API, which then assigns these proof tasks.

### Vortex Hub

The Vortex Hub is responsible for broadcasting proof tasks. It broadcasts tasks to the Vortex network, and provers who respond within a certain timeframe randomly receive proof generation tasks. The Vortex Hub also handles user proof requests; users must declare their tasks to the Vortex Hub before sending the specific content and circuitry needed to generate proofs, after which the Vortex Hub broadcasts these tasks within the Vortex network.

### Vortex Network

The Vortex Network consists of Vortex nodes operated by users, whose job is to generate ZK-SNARK proofs and return them via the Vortex API.

## Introduction to Vortex GPU Confidential Computing Technology

### Confidential Computing(CC) Working Mode of Confidential GPU

![image](https://github.com/Mirror-Tang/vortex/blob/master/image/1.png)

• The CC work mode must be configured before the GPU starts. In cloud scenarios, it is set by the VMM; the CC work mode bit is configured into the GPU EEPROM, and then a GPU reset must be performed for it to take effect.
• The GPU attestation report records under which CC work mode the GPU is operating. This ensures that an untrusted hypervisor can only set the CC work mode as intended.
• A GPU reset will clear the GPU memory and state, including session keys and secrets.

### How to Protect Memory after Enabling CC Work Mode？

![image](https://github.com/Mirror-Tang/vortex/blob/master/image/2.png)

• Most of the GPU memory is configured into the Compute Protected Region (CPR), where the memory is protected by hardware firewalls internal to the GPU.
• A small portion of the GPU memory does not require protection:Used to store encrypted CUDA command buffers; Used for encrypted bounce buffers for NVLINK end-to-end communication.
• In the CVM, the NV driver commands the GPU to allocate bounce buffers in the unprotected memory area and maps them to the shared memory area of the CPU.
• Subsequently, the NV driver uses the SPDM session key to write encrypted data into the bounce buffer.

### How does Confidential Computing Protect CUDA Programs (without needing to change the CUDA programs)?

![image](https://github.com/Mirror-Tang/vortex/blob/master/image/3.png)

• All communication between the CPU and GPU is encrypted, including data transfers, commands, and CUDA kernels.
• Due to the IOMMU prohibiting devices from directly accessing CVM(Confidential Virtual Machine) private memory, a bounce buffer located in the shared memory is required for data exchange between the CPU and GPU.
• When sending data to the GPU device:The NVIDIA driver within the CVM encrypts the data to be sent and then writes it into the bounce buffer;The GPU DMA engine reads the encrypted data from the bounce buffer and decrypts it into the GPU's protected memory.
• When reading data from the GPU device:The GPU DMA engine writes the encrypted data into the bounce buffer via the PCIe bus;The NVIDIA driver within the CVM reads the encrypted data from the bounce buffer and decrypts it into the CVM's private memory.

### Internal Protection Mechanisms of a Confidential GPU

![image](https://github.com/Mirror-Tang/vortex/blob/master/image/4.png)

If the GPU is started in CC (Confidential Computing) work mode, it blocks all inbound and outbound access to the GPU CPR (Compute Protected Region) memory.
• The PCIe Firewall is responsible for blocking:Access from the CPU side to the majority of GPU registers; Any access to GPU CPR memory.
• The NVLink Firewall is responsible for blocking any access to GPU CPR memory from peer GPUs.
• The DMA engine can only read or write to memory outside of the GPU CPR.
• Block all other engines (such as compute SMs) from accessing memory outside of the CPR.
After CC work mode is enabled, all GPU performance counters are disabled to protect against side-channel attacks.

### Multi GPU Secret Computing Based on CVM

![image](https://github.com/Mirror-Tang/vortex/blob/master/image/5.png)

• Enable NVLink communication between GPUs and devices. 1. After enabling CC working mode, do not use PCIe P2P; 2. Both CUDA APIs and hardware firewalls prohibit direct pointer dereference of peer GPU memory.
• The DMA engine on the source and destination GPUs uses a shared session key to protect transmission on NVLink.CudaMemcpyDeviceToDevice() transmits encrypted data through a bounce buffer.1. The DMA engine on the source GPU side is responsible for encrypting data, and then transmitting the encrypted data to the unprotected memory on the source GPU side through an untrusted NVLink; 2. The DMA engine on the destination GPU side is responsible for decrypting encrypted data from the bounce buffer into its own GPU protected memory.

### Confidential MIG

![image](https://github.com/Mirror-Tang/vortex/blob/master/image/6.png)

Related Technologies:

• NVIDIA vGPU: Allows multiple VM instances to share a single NVIDIA GPU Physical Function (PF).
• Multi-Instance GPU (MIG): Divides GPU resources into multiple GPU instances, which can be considered as multiple sub-GPUs.
• SR-IOV: Exposes GPU Virtual Functions (VF) as PCIe devices and passes them through to VMs.

### Security Features Dependent on GPU TEE

• On-chip RoT (Root of Trust): RoT is responsible for ensuring the integrity of software and hardware during device startup and operation.
• NVIDIA RISC-V Microcontroller: Handles specific security tasks and operations such as key management and secure boot.
• Encrypted Firmware: Firmware is encrypted during storage or transmission to prevent leakage or unauthorized modifications.
• FIPS 140-3 Level 2 Cryptographic Encryption: Used to prevent unauthorized access to encryption keys.
• Device Certificates: Digital certificates that prove device identity, used for authentication and secure communication on networks.
• Secure Boot: Ensures that the device only loads and executes verified, trustworthy operating systems and software.
• Measured Boot: A mechanism that guarantees the security of the boot process by measuring (recording and verifying) the integrity of each component loaded during the boot process, ensuring the device starts from a trusted baseline state.
• Hardware Fault Injection Counter: Used to resist attacks that attempt to tamper with or bypass security mechanisms through maliciously induced hardware faults (such as voltage, temperature changes).
• Firmware Revocation: Used to revoke or withdraw firmware versions that are outdated or known to have security vulnerabilities to ensure device safety.
• SR-IOV for Secure MIG: Used for single GPU multi-VM setups to ensure isolation and security in resource usage for each virtual GPU instance.
• PKC Firmware Authentication: Uses public key cryptography to verify the integrity and origin of firmware, ensuring it has not been tampered with.
• Unique Identity Key Pair: Refers to a device having a unique pair of public and private keys used for various security functions, such as encrypted communication, digital signatures, etc.

## RISC ZERO Remote Proof Generation

### Overview of the Prove Process

1. Users initiate a new Prove task by sending a POST request to the ProveAPI containing the contents of the `input_and_elf.json` file. As a response, the API returns a `task_id` to the user, which serves as a reference for tracking the task's progress and execution status.
2. The API receives `input` and `elf` and forwards them to the Prove side.
3. The Prove side reads `elf` and `input`, using GPU acceleration to generate a proof (the `receipt.json` file). The `receipt.json` file is then returned to the task scheduling system. This system associates the `receipt.json` file and the corresponding execution results with the previously generated `task_id`, and persists `task_id`, `receipt`, `elf`, and `input` for future queries.
4. The Prove side returns the `receipt` to the user via the API. Users can verify the validity of the proof or perform other related operations based on the information in the `receipt`.

### Generating Configuration File input_and_elf.json

Users need to generate a configuration file containing inputs and elf named input_and_elf.json locally. The specific steps are as follows:

1. Users should first replace the lib.rs file in the risc-build package with the new lib.rs file.

If the user's current project was created with `cargo risczero new my_project  --guest-name my_guest`, the path of the lib.rs file in this project is located at ~/.cargo/registry/src/your-rust-mirror/risc0-build-0.xx/src/lib.rs.

If the user's current project is cloned from risc0, the path of the lib.rs file in this project is located at /risc0/build/src/lib.rs.

2. After replacing the file, users also need to transfer the circuit inputs originally built in /host to /methods/src/build.rs and supplement the following code in build.rs:

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

Here is an example:

Taking variables a and b as two inputs for the circuit, both need to be converted into Vec<u32> form through the to_vec() function and encapsulated as Vec<Vec<u32>> type to be passed into risc0_build::embed_methods(). The constructed example code for build.rs is as follows:

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

3. Accordingly, users should add dependencies for risc0-zkvm and serde in the Cargo.toml located in the same directory as build.rs. Other dependencies depend on the specific inputs users wish to build, and users need to supplement them on their own.

4. Next, users execute cargo build in the /methods directory, which will generate the input_and_elf.json file in the /target path of the project root directory, containing the circuit inputs and the elf encoding.

input_and_elf.json Example:

```
{
  "input": [
    [17, 0],
    [23, 0]
  ],
  "elf": [
    127,
    ...,
    0
  ]
}
```

After locally generating the input_and_elf.json configuration file, users need to interact with our API. We can create a new Prove task for users, and users can also track the status of their Prove tasks.

# Create a New Prove Task

## Overview

This interface is used to create a new Prove task. Users create a task by sending a POST request to `api/v1/prove/create`. Upon successful creation, the interface returns a `task_id` that users can use to query or manage the Prove task.

## Request

- **Method:** POST
- **URL:** `/api/v1/prove/create`
- **Content-Type:** `application/json`
  
### Request Parameters

Provided via the POST body:

- **input** - Type: `str` - The input for the circuit.
- **elf** - Type: `str` - The binary file produced after circuit compilation, encoded as u8.

## Response

### Response Body

The response is in JSON format and includes the following fields:

- **code:** Status code
- **msg:** Message
- **results:** The unique identifier for the newly created Prove task.

### Successful Response Example

```json
{
 "code": 0,
 "msg": "Successfully",
 "results": {}
}
```

### Response Status Codes

- **200 OK:** The request was successful, and the Prove task has been created.
- **400 Bad Request:** The request format is incorrect or missing necessary information.
- **401 Unauthorized:** The user is not authenticated, and the request is denied.
- **500 Internal Server Error:** There was an internal server error, and the request could not be completed.

# Query the Execution Status of a Prove Task

## Overview

This interface allows users to query the current execution status of a specified Prove task. By sending a GET request to `api/v1/prove/query` with the task's `task_id`, users can obtain the task's status information.

## Request

- **Method:** GET
- **URL:** `/api/v1/prove/query`
- **Content-Type:** `application/json`
  
### Request Parameters

Provided via URL parameters:

- **task_id** - Type: `str` - The unique identifier of the Prove task whose status is being queried.

## Response

### Response Body

The response is in JSON format and includes the following fields:

- **code:** Status code
- **msg:** Message
- **results:** Receipt

### Successful Response Example

```json
{
 "code": 0,
 "msg": "Successfully",
 "results": {}
}
```

### Response Status Codes

- **200 OK:** The request was successful, and the task status was successfully returned.
- **400 Bad Request:** The request format is incorrect or missing necessary information.
- **401 Unauthorized:** The user is not authenticated, and the request is denied.
- **500 Internal Server Error:** There was an internal server error, and the request could not be completed.

## Circom Remote Proof Generation

## HALO2 Remote Proof Generation

## Gnark Remote Proof Generation

## Vortex Private Cloud

Vortex Private Cloud is a solution that helps manage local private server clusters (primarily GPU servers and computing-purpose CPU servers) and offers computing services to multiple users in a containerized manner. The main differences from conventional private cluster management software include:

1. Low Entry Barrier and Maintenance. Local servers only need to deploy a few simple service components, resulting in very low maintenance efforts. Other functions like member user management, registration and login, control plane-related services, and computing power scheduling are all provided by the cloud. This means you don't need additional machines for deploying management services, databases, or message queues to have a complete set of cluster management software infrastructure.

2. Excellent User Experience. Unlike private deployments, deployment does not mean that features remain unchanged. Instead, cloud control plane functionalities will continuously upgrade to meet ongoing and diverse experience and feature needs (your requests are welcome). Upgrades and maintenance processes are automated, requiring no user intervention.

3. High Scalability. Only GPU servers are required to complete a closed-loop cluster management function, while also allowing for expansion of your cluster features based on other local facilities you may provide, such as shared file storage and image repositories.

4. Mobile Support. Supports mobile device management of instances (such as starting or stopping instances) at any time and from anywhere, with push notifications for host and instance anomalies.

5. High Data Security. Offers nearly the same data security guarantees as fully private deployment options. Since the cloud only manages control plane data such as user management, computing power inventory, and scheduling, and server access can be restricted to internal network environments only, it simultaneously supports low maintenance, great user experience, and high security.

6. Based on AutoDL, Vortex Private Cloud seamlessly utilizes the CodeWithGPU community images.

---
