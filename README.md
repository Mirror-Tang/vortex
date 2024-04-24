# vortex
Modular ZK proof layer

---

We can provide users with a fast, secure, and efficient zero-knowledge proof generation service, known as the Prove service, based on distributed computing resources. Users only need to upload the necessary files (input_and_elf.json) and data to our Prove endpoint through our provided API interface. Leveraging our powerful backend GPU computing capabilities, we can generate the required zero-knowledge proofs for users accurately and in a very short time.

# Overview of the Prove Process

1. Users initiate a new Prove task by sending a POST request to the ProveAPI containing the contents of the `input_and_elf.json` file. As a response, the API returns a `task_id` to the user, which serves as a reference for tracking the task's progress and execution status.
2. The API receives `input` and `elf` and forwards them to the Prove side.
3. The Prove side reads `elf` and `input`, using GPU acceleration to generate a proof (the `receipt.json` file). The `receipt.json` file is then returned to the task scheduling system. This system associates the `receipt.json` file and the corresponding execution results with the previously generated `task_id`, and persists `task_id`, `receipt`, `elf`, and `input` for future queries.
4. The Prove side returns the `receipt` to the user via the API. Users can verify the validity of the proof or perform other related operations based on the information in the `receipt`.

# Generating Configuration File input_and_elf.json

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
 "msg": "Successfully created.",
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
 "msg": "Successfully.",
 "results": {}
}
```

### Response Status Codes

- **200 OK:** The request was successful, and the task status was successfully returned.
- **400 Bad Request:** The request format is incorrect or missing necessary information.
- **401 Unauthorized:** The user is not authenticated, and the request is denied.
- **500 Internal Server Error:** There was an internal server error, and the request could not be completed.

---
