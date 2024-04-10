# vortex
Modular ZK proof layer

---

# Overview of the Prove Process

1. Users initiate a new Prove task by sending a POST request to the ProveAPI containing the contents of the `input_and_elf.json` file. As a response, the API returns a `task_id` to the user, which serves as a reference for tracking the task's progress and execution status.
2. The API receives `input` and `elf` and forwards them to the Prove side.
3. The Prove side reads `elf` and `input`, using GPU acceleration to generate a proof (the `receipt.json` file). The `receipt.json` file is then returned to the task scheduling system. This system associates the `receipt.json` file and the corresponding execution results with the previously generated `task_id`, and persists `task_id`, `receipt`, `elf`, and `input` for future queries.
4. The Prove side returns the `receipt` to the user via the API. Users can verify the validity of the proof or perform other related operations based on the information in the `receipt`.

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
