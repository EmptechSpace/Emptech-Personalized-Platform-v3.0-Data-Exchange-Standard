# Production Management System API Documentation - Personalization System Integration

> **Document Version**: v1.0  
> **Target Audience**: Personalization System Developers  
> **Protocol**: HTTP RESTful + JSON  
> **Authentication Method**: Basic Auth + Bearer Token

---

## Table of Contents

1. [API Specification Guidelines](#1-api-specification-guidelines)
2. [Authentication API](#2-authentication-api)
3. [Task Retrieval API](#3-task-retrieval-api)
4. [Batch Confirmation API](#4-batch-confirmation-api)
5. [Batch Feedback API](#5-batch-feedback-api)
6. [Heartbeat and Status API](#6-heartbeat-and-status-api)
7. [Error Code Reference](#7-error-code-reference)
8. [Appendix](#8-appendix)

---

## 1. API Specification Guidelines

### 1.1 API Path Conventions

| API Type | Path Prefix | Example |
|----------|-------------|---------|
| Authentication API | `/open-api/mes/auth/**` | `/open-api/mes/auth/token` |
| Business API | `/open-api/mes/personalization/**` | `/open-api/mes/personalization/batches/pending` |

### 1.2 Standard Request and Response Specification

#### Standard Request Headers

| Parameter | Required | Description |
|-----------|----------|-------------|
| Authorization | Yes | `Bearer {accessToken}` |
| X-Line-ID | Yes | Unique production line identifier `lineId` |
| Content-Type | Yes | `application/json` |

#### Standard Response Body Structure

```json
{
  "code": 0,
  "msg": "success",
  "data": { }
}
```

| Field | Type | Description |
|-------|------|-------------|
| code | int | Business status code. **0 indicates success**, other values indicate errors. |
| msg | string | Status description or error description. |
| data | object | Business data payload. `null` in case of failure. |

**Error Code Explanation:**
- `code = 0`: Business operation successful.
- `code > 0`: Business exception with custom error code. The `msg` field contains the error description.

### 1.3 Business Identifier Conventions

| Identifier | Description | Example |
|------------|-------------|---------|
| lineId | Unique production line identifier | LINE_001 |
| deviceSn | Unique device identifier | DEV_LINE001_001 |
| orderSn | Production order number | ORDER20240410001 |
| planSn | Production task plan number | PLAN20240410001 |
| batchSn | Batch number (production line level) | BATCH20240410001_LINE001 |
| businessSn | Unique production data identifier (single card) | BSN2024041000001 |
| docSn | Document/Certificate number | B1231341 |

**Relationship Description:**

- One production line (`lineId`) is bound to one device (`deviceSn`).
- One batch (`batchSn`) corresponds to one production line.
- One batch contains multiple production data entries (`businessSn`).

### 1.4 Data Package Type Specification

| dataPathType | Description | Handling Method |
|--------------|-------------|-----------------|
| S3 | MinIO Object Storage | Personalization system directly uses pre-signed URL to pull from MinIO. |
| FTP | FTP File Server | Personalization system downloads file from FTP server. |
| LOCAL | Local File System | Personalization system retrieves file from specified local path. |

---

## 2. Authentication API

### 2.1 Obtain Access Token

Use Basic Authentication to obtain an access token. The authentication process verifies the binding relationship between the production line and the device.

```http
POST /open-api/mes/auth/token
```

**Request Headers:**

```
Authorization: Basic {base64(clientId:clientSecret)}
Content-Type: application/json
```

**Request Body:**

```json
{
  "lineId": "LINE_001",
  "deviceSn": "DEV_LINE001_001",
  "deviceInfo": {
    "deviceType": "PERSONALIZATION_MACHINE",
    "softwareVersion": "v2.1.0",
    "hardwareVersion": "v1.0"
  }
}
```

**Parameter Description:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| lineId | string | Yes | Unique production line identifier. |
| deviceSn | string | Yes | Unique device identifier, must be bound to the production line. |
| deviceInfo.deviceType | string | Yes | Device type. |
| deviceInfo.softwareVersion | string | No | Software version number (redundant). |
| deviceInfo.hardwareVersion | string | No | Hardware version number (redundant). |

**Response Example (Success):**

```json
{
  "code": 0,
  "msg": "Authentication successful",
  "data": {
    "accessToken": "eyJhbGciOiJSUzI1NiJ9...",
    "tokenType": "Bearer",
    "expiresIn": 7200,
    "expireTime": "2024-04-10T11:00:00Z",
    "lineId": "LINE_001",
    "deviceSn": "DEV_LINE001_001",
    "lineInfo": {
      "lineId": "LINE_001",
      "lineName": "Production Line 1"
    }
  }
}
```

**Response Example (Failure):**

```json
{
  "code": 10001,
  "msg": "Device does not match production line",
  "data": null
}
```

---

## 3. Task Retrieval API

### 3.1 Get Batch Task List

Retrieves the list of pending **batch tasks** for the current production line, sorted by priority and creation time.

```http
POST /open-api/mes/personalization/batches/pending
```

**Request Headers:**

```
Authorization: Bearer {accessToken}
X-Line-ID: LINE_001
Content-Type: application/json
```

**Request Body:**

```json
{
  "limit": 10
}
```

**Parameter Description:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| limit | int | No | 10 | Number of batches to return, maximum 50. |

**Response Example (Success):**

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "total": 5,
    "limit": 10,
    "batches": [
      {
        "orderSn": "ORDER20240410001",
        "planSn": "PLAN20240410001",
        "batchSn": "BATCH20240410001_LINE001",
        "productType": "IDCARD",
        "productName": "Resident ID Card",
        "priority": 1,
        "taskCount": 100,
        "deadline": "2024-04-10T18:00:00Z",
        "createTime": "2024-04-10T08:00:00Z"
      },
      {
        "orderSn": "ORDER20240410002",
        "planSn": "PLAN20240410002",
        "batchSn": "BATCH20240410002_LINE001",
        "productType": "IDCARD",
        "productName": "Resident ID Card",
        "priority": 2,
        "taskCount": 50,
        "deadline": "2024-04-10T20:00:00Z",
        "createTime": "2024-04-10T09:00:00Z"
      }
    ]
  }
}
```

**Response Field Description:**

| Field Path | Type | Description |
|------------|------|-------------|
| orderSn | string | Production order number. |
| planSn | string | Production task plan number. |
| batchSn | string | Batch number (production line level). |
| productType | string | Product type. |
| productName | string | Product name. |
| priority | int | Priority, smaller number indicates higher priority. |
| taskCount | int | Number of tasks within this batch. |
| deadline | string | Due date. |
| createTime | string | Batch creation time. |

---

### 3.2 Get Task Details List for a Batch

Retrieves detailed information for all pending production data under a specific batch number (`batchSn`), **supports pagination**.

```http
POST /open-api/mes/personalization/batches/tasks
```

**Request Headers:**

```
Authorization: Bearer {accessToken}
X-Line-ID: LINE_001
Content-Type: application/json
```

**Request Body:**

```json
{
  "batchSn": "BATCH20240410001_LINE001",
  "page": 1,
  "pageSize": 50
}
```

**Parameter Description:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| batchSn | string | Yes | - | Batch number. |
| page | int | No | 1 | Page number. |
| pageSize | int | No | 50 | Number of items per page, maximum 100. |

**Response Example (Success):**

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "batchSn": "BATCH20240410001_LINE001",
    "page": 1,
    "pageSize": 50,
    "total": 100,
    "totalPages": 2,
    "tasks": [
      {
        "orderSn": "ORDER20240410001",
        "planSn": "PLAN20240410001",
        "batchSn": "BATCH20240410001_LINE001",
        "businessSn": "BSN2024041000001",
        "docSn": "CARD20240001",
        "productType": "IDCARD",
        "productName": "Resident ID Card",
        "priority": 1,
        "deadline": "2024-04-10T18:00:00Z",
        "createTime": "2024-04-10T08:00:00Z",
        "dataInfo": {
          "dataPathType": "S3",
          "dataPackageUrl": "https://minio.example.com/prepared-data/BATCH20240410001_LINE001/BSN2024041000001.dat?X-Amz-Algorithm=...",
          "dataProxyPath":"/prepared-data/BATCH20240410001_LINE001/BSN2024041000001.dat",
          "expireTime": "2024-04-10T10:00:00Z",
          "checksum": "sha256:a1b2c3d4e5f6...",
          "size": 2097152
        }
      },
      {
        "orderSn": "ORDER20240410001",
        "planSn": "PLAN20240410001",
        "batchSn": "BATCH20240410001_LINE001",
        "businessSn": "BSN2024041000002",
        "docSn": "CARD20240002",
        "productType": "IDCARD",
        "productName": "Resident ID Card",
        "priority": 1,
        "deadline": "2024-04-10T18:00:00Z",
        "createTime": "2024-04-10T08:00:00Z",
        "dataInfo": {
          "dataPathType": "S3",
          "dataPackageUrl": "https://minio.example.com/prepared-data/BATCH20240410001_LINE001/BSN2024041000002.dat?X-Amz-Algorithm=...",
          "expireTime": "2024-04-10T10:00:00Z",
          "checksum": "sha256:b2c3d4e5f6g7...",
          "size": 2097152
        }
      }
    ]
  }
}
```

**dataInfo Field Description:**

| Field Path | Type | Description |
|------------|------|-------------|
| dataPathType | string | Data path type: S3/FTP/LOCAL. |
| dataPackageUrl | string | Data package download URL (pre-signed URL or file path). |
| dataProxyPath | string | Data package proxy address, only for LOCAL type. Used when network is poor, copy data package in advance. |
| expireTime | string | Download URL expiration time (S3/FTP only). |
| checksum | string | Data package checksum (SHA-256). |
| size | long | Data package size (bytes). |

**dataPathType Description:**

| dataPathType | dataPackageUrl Format Example |
|--------------|-------------------------------|
| S3 | `https://minio.example.com/...` (Pre-signed URL) |
| FTP | `ftp://ftp.example.com/...` or `ftps://...` |
| LOCAL | `/data/prepared/BATCH20240410001_LINE001/BSN2024041000001.dat` |

---

## 4. Batch Confirmation API

### 4.1 Confirm Batch Start Processing

Before the personalization system starts processing a batch, it must call this API to confirm the batch start. **Confirmation is performed at the batch level, not for individual businessSn.**

```http
POST /open-api/mes/personalization/batches/confirm
```

**Request Headers:**

```
Authorization: Bearer {accessToken}
X-Line-ID: LINE_001
Content-Type: application/json
Idempotency-Key: {uuid}
```

**Request Body:**

```json
{
  "batchSn": "BATCH20240410001_LINE001",
  "confirmTime": "2024-04-10T09:00:00Z",
  "expectedCompletionTime": "2024-04-10T12:00:00Z",
  "operatorId": "OP001",
  "operatorName": "Operator Zhang San"
}
```

**Parameter Description:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| batchSn | string | Yes | Batch number (production line level). |
| confirmTime | string | Yes | Confirmation time (ISO 8601 format). |
| expectedCompletionTime | string | No | Estimated completion time. |
| operatorId | string | No | Operator ID. |
| operatorName | string | No | Operator name. |

**Response Example:**

```json
{
  "code": 0,
  "msg": "Batch confirmation successful",
  "data": {
    "batchSn": "BATCH20240410001_LINE001",
    "status": "PROCESSING",
    "confirmTime": "2024-04-10T09:00:00Z",
    "updateTime": "2024-04-10T09:00:01Z"
  }
}
```

**Response Field Description:**

| Field | Type | Description |
|-------|------|-------------|
| batchSn | string | Batch number. |
| status | string | Batch status: PROCESSING. |
| confirmTime | string | Confirmation time. |
| updateTime | string | Update time. |

---

## 5. Batch Feedback API

### 5.1 Submit Batch Card Production Results

After card production is complete, use this API to submit the results for the entire batch. **Supports submitting results for the whole batch, including detailed attempt records and consumable information.**

<span style="color:red">For the batch feedback interface, considering the data volume of a batch, multiple feedback rounds are supported. The client handles sharding logic, and the server considers the submission complete when the count of `businessSn` reaches the batch total.</span>

```http
POST /open-api/mes/personalization/batches/feedback
```

**Request Headers:**

```
Authorization: Bearer {accessToken}
X-Line-ID: LINE_001
Content-Type: application/json
Idempotency-Key: {uuid}
```

**Request Body:**

```json
{
  "feedbackId": "FB20240410001",
  "batchSn": "BATCH20240410001_LINE001",
  "submitTime": "2024-04-10T12:00:00Z",
  "summary": {
    "totalCount": 100,
    "successCount": 98,
    "failCount": 2,
    "wasteCount": 4,
    "totalAttempts": 105
  },
  
  "results": [
    {
      "businessSn": "BSN2024041000001",
      "docSn": "CARD20240001",
      "status": "SUCCESS",
      "attemptCount": 1,
      "personalizeTime": "2024-04-10T10:05:00Z",
      
      "attempts": [
        {
          "attemptNo": 1,
          "status": "SUCCESS",
          "startTime": "2024-04-10T10:00:00Z",
          "endTime": "2024-04-10T10:05:00Z",
          "materials": [
            { "type": "CHIP", "id": "CHIP001" },
            { "type": "PVC", "id": "PVC001" },
            { "type": "RIBBON", "id": "RIBBON001" }
          ]
        }
      ]
    },
    {
      "businessSn": "BSN2024041000002",
      "docSn": "CARD20240002",
      "status": "SUCCESS",
      "attemptCount": 3,
      "personalizeTime": "2024-04-10T10:35:00Z",
      
      "attempts": [
        {
          "attemptNo": 1,
          "status": "FAILED",
          "startTime": "2024-04-10T10:10:00Z",
          "endTime": "2024-04-10T10:12:00Z",
          "errorCode": "ERR_CHIP_WRITE_FAIL",
          "errorMessage": "Chip write failed",
          "isWaste": true,
          "wasteType": "CHIP_FAILED",
          "materials": [
            { "type": "CHIP", "id": "CHIP002", "wasted": true },
            { "type": "PVC", "id": "PVC002", "wasted": true },
            { "type": "RIBBON", "id": "RIBBON001", "wasted": false }
          ]
        },
        {
          "attemptNo": 2,
          "status": "FAILED",
          "startTime": "2024-04-10T10:20:00Z",
          "endTime": "2024-04-10T10:23:00Z",
          "errorCode": "ERR_PRINT_QUALITY",
          "errorMessage": "Print quality failed",
          "isWaste": true,
          "wasteType": "PRINT_FAILED",
          "materials": [
            { "type": "CHIP", "id": "CHIP003", "wasted": true },
            { "type": "PVC", "id": "PVC003", "wasted": true },
            { "type": "RIBBON", "id": "RIBBON002", "wasted": true }
          ]
        },
        {
          "attemptNo": 3,
          "status": "SUCCESS",
          "startTime": "2024-04-10T10:30:00Z",
          "endTime": "2024-04-10T10:35:00Z",
          "materials": [
            { "type": "CHIP", "id": "CHIP004" },
            { "type": "PVC", "id": "PVC004" },
            { "type": "RIBBON", "id": "RIBBON003" }
          ]
        }
      ],
      
      "wasteSummary": {
        "totalWaste": 2,
        "wasteMaterials": {
          "chips": ["CHIP002", "CHIP003"],
          "pvcSheets": ["PVC002", "PVC003"],
          "ribbons": ["RIBBON002"]
        }
      }
    }
  ]
}
```

**Parameter Description:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| feedbackId | string | Yes | Unique feedback identifier for idempotency control. |
| batchSn | string | Yes | Batch number. |
| submitTime | string | Yes | Submission time. |
| summary | object | Yes | Batch summary information. |
| summary.totalCount | int | Yes | Total number in batch. |
| summary.successCount | int | Yes | Number of successes. |
| summary.failCount | int | Yes | Number of failures. |
| summary.wasteCount | int | Yes | Number of wasted items. |
| summary.totalAttempts | int | Yes | Total number of attempts. |
| results | array | Yes | List of per-card production results. |

**results Item Field Description:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| businessSn | string | Yes | Unique production data identifier. |
| docSn | string | Yes | Document/Certificate number. |
| status | string | Yes | Production result: SUCCESS/FAILED. |
| attemptCount | int | Yes | Number of attempts. |
| personalizeTime | string | Yes | Production completion time. |
| attempts | array | Yes | Detailed information for each attempt. |
| wasteSummary | object | No | Waste summary information (if multiple attempts failed). |

**attempts Item Field Description:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| attemptNo | int | Yes | Attempt sequence number. |
| status | string | Yes | Attempt result: SUCCESS/FAILED. |
| startTime | string | Yes | Attempt start time. |
| endTime | string | Yes | Attempt end time. |
| errorCode | string | No | Error code (required on failure). |
| errorMessage | string | No | Error description (optional on failure). |
| isWaste | boolean | No | Whether waste was generated. |
| wasteType | string | No | Waste type. |
| materials | array | Yes | Consumable usage information. |

**materials Item Field Description:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| type | string | Yes | Consumable type: CHIP/PVC/RIBBON. |
| id | string | Yes | Unique consumable identifier. |
| wasted | boolean | No | Whether the consumable was wasted (default false). |

---

## 6. Heartbeat and Status API

### 6.1 Report Heartbeat and Status

The personalization system reports heartbeat and current status periodically (recommended every 30 seconds). The server responds with an indicator of whether new tasks are available.

```http
POST /open-api/mes/personalization/heartbeat
```

**Request Headers:**

```
Authorization: Bearer {accessToken}
X-Line-ID: LINE_001
Content-Type: application/json
```

**Request Body:**

```json
{
  "lineId": "LINE_001",
  "deviceSn": "DEV_LINE001_001",
  "status": "RUNNING",
  "currentBatch": {
    "orderSn": "ORDER20240410001",
    "planSn": "PLAN20240410001",
    "batchSn": "BATCH20240410001_LINE001"
  },
  "progress": {
    "completed": 50,
    "total": 100,
    "percentage": 50.0
  },
  "reportTime": "2024-04-10T10:30:00Z"
}
```

**Response Example:**

```json
{
  "code": 0,
  "msg": "Heartbeat reported successfully",
  "data": {
    "serverTime": "2024-04-10T10:30:01Z",
    "nextHeartbeatInterval": 30,
    "hasNewTask": true,
    "newTaskCount": 2
  }
}
```

**Heartbeat Response Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| hasNewTask | boolean | Indicates if new tasks are pending. |
| newTaskCount | int | Number of new tasks (optional). |
| nextHeartbeatInterval | int | Suggested interval for the next heartbeat (seconds). |

**Business Logic Description:**

Heartbeat processing logic for the personalization system:
1. Send heartbeat request periodically (e.g., every 30 seconds).
2. Check the `hasNewTask` field in the response.
3. If `hasNewTask = true`, immediately call the **[3.1 Get Batch Task List](#31-get-batch-task-list)** API to retrieve tasks.
4. Process tasks based on the retrieved list.

---

## 7. Error Code Reference

### 7.1 HTTP Status Codes

| Status Code | Description | Suggested Action |
|-------------|-------------|------------------|
| 200 | Success | Process response data normally. |
| 400 | Bad Request | Check request parameters against specification. |
| 401 | Unauthorized | Token invalid or expired, re-authenticate. |
| 403 | Forbidden | No permission to access the resource. |
| 404 | Not Found | Check if the requested resource ID is correct. |
| 409 | Conflict | Resource status conflict. |
| 429 | Too Many Requests | Rate limit triggered, reduce request frequency. |
| 500 | Internal Server Error | Server exception, contact administrator. |
| 503 | Service Unavailable | Service temporarily unavailable, retry later. |

### 7.2 Business Error Codes (code > 0)

| Error Code | Description | Suggested Action |
|------------|-------------|------------------|
| **Authentication Related** |
| 10001 | Authentication failed | Check clientId and clientSecret. |
| 10002 | Token expired | Use refreshToken or re-authenticate. |
| 10003 | Invalid token | Re-authenticate. |
| 10004 | Production line not authorized | Contact admin to configure line permissions. |
| 10005 | Device does not match production line | Check if deviceSn is bound to lineId. |
| **Task Related** |
| 20001 | Batch does not exist | Check if batchSn is correct. |
| 20002 | Batch already assigned | Batch is occupied by another device, pull another batch. |
| 20003 | Illegal batch status | Operation not allowed in current status. |
| 20004 | Batch timed out | Batch exceeded max processing time, automatically released. |
| 20005 | Production line mismatch with batch | Batch does not belong to current line. |
| **Data File Related** |
| 30001 | Data package does not exist | Check dataPackageUrl or contact admin. |
| 30002 | Data package download link expired | Re-fetch task details for a new download link. |
| 30003 | Data package checksum failed | Data may be corrupted, re-download or contact admin. |
| 30004 | Data path type not supported | Check if dataPathType is S3/FTP/LOCAL. |
| **Feedback Related** |
| 40001 | Feedback already received | Duplicate submission detected. |
| 40002 | Batch identifier does not exist | Check if batchSn is correct. |
| 40003 | Illegal state transition | Cannot update to target status from current status. |
| 40004 | Invalid production result data | Check productionResult field integrity. |

### 7.3 Card Production Error Codes (Used in feedback errorCode field)

| Error Code | Category | Description | Retryable | Retry Strategy |
|------------|----------|-------------|-----------|----------------|
| ERR_CHIP_WRITE_FAIL | Chip Error | Chip write failed | Yes | Replace chip and retry. |
| ERR_CHIP_READ_FAIL | Chip Error | Chip read failed | Yes | Retry reading. |
| ERR_CHIP_VERIFY_FAIL | Chip Error | Chip verification failed | No | Mark as failed. |
| ERR_PRINTER_JAM | Print Error | Printer jam | Yes | Clear jam and retry. |
| ERR_PRINT_QUALITY | Print Error | Print quality failed | No | Mark as failed. |
| ERR_CARD_FEED_FAIL | Mechanical Error | Card feed failed | Yes | Retry feeding card. |
| ERR_CARD_EJECT_FAIL | Mechanical Error | Card eject failed | Yes | Manual intervention required. |
| ERR_DATA_CHECK_FAIL | Data Error | Data verification failed | No | Mark as failed. |
| ERR_DEVICE_OFFLINE | Device Error | Device offline | Yes | Retry after device recovers. |

---

## 8. Appendix

### 8.1 Complete Business Process Sequence Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            Business Process                             │
└─────────────────────────────────────────────────────────────────────────┘

1. Authentication to obtain token
   
   POST /open-api/mes/auth/token
   Authorization: Basic {base64(clientId:clientSecret)}
   
   Request:
   {
     "lineId": "LINE_001",
     "deviceSn": "DEV_LINE001_001",
     "deviceInfo": {
       "deviceType": "PERSONALIZATION_MACHINE",
       "softwareVersion": "v2.1.0"
     }
   }
   
   ──────────────────────────────────────────────▶
   
   Response:
   {
     "code": 0,
     "data": {
       "accessToken": "eyJhbGciOiJSUzI1NiJ9...",
       "expiresIn": 7200,
       "expireTime": "2024-04-10T11:00:00Z",
       "lineId": "LINE_001",
       "deviceSn": "DEV_LINE001_001"
     }
   }
   
   ◀──────────────────────────────────────────────

2. Periodic heartbeat reporting (recommended every 30 seconds)
   
   POST /open-api/mes/personalization/heartbeat
   Authorization: Bearer {accessToken}
   X-Line-ID: LINE_001
   
   Request:
   {
     "lineId": "LINE_001",
     "deviceSn": "DEV_LINE001_001",
     "status": "IDLE",
     "reportTime": "2024-04-10T10:00:00Z"
   }
   
   ──────────────────────────────────────────────▶
   
   Response:
   {
     "code": 0,
     "data": {
       "hasNewTask": true,
       "newTaskCount": 2,
       "nextHeartbeatInterval": 30
     }
   }
   
   ◀──────────────────────────────────────────────

3. [Conditional] If hasNewTask = true, query pending batch list
   
   POST /open-api/mes/personalization/batches/pending
   Authorization: Bearer {accessToken}
   X-Line-ID: LINE_001
   
   Request:
   {
     "limit": 10
   }
   
   ──────────────────────────────────────────────▶
   
   Response:
   {
     "code": 0,
     "data": {
       "total": 5,
       "batches": [
         {
           "orderSn": "ORDER20240410001",
           "planSn": "PLAN20240410001",
           "batchSn": "BATCH20240410001_LINE001",
           "productType": "IDCARD",
           "taskCount": 100,
           "createTime": "2024-04-10T08:00:00Z"
         }
       ]
     }
   }
   
   ◀──────────────────────────────────────────────

4. Get task details list for a batch (get data package download info)
   
   POST /open-api/mes/personalization/batches/tasks
   Authorization: Bearer {accessToken}
   X-Line-ID: LINE_001
   
   Request:
   {
     "batchSn": "BATCH20240410001_LINE001",
     "page": 1,
     "pageSize": 50
   }
   
   ──────────────────────────────────────────────▶
   
   Response:
   {
     "code": 0,
     "data": {
       "batchSn": "BATCH20240410001_LINE001",
       "page": 1,
       "pageSize": 50,
       "total": 100,
       "totalPages": 2,
       "tasks": [
         {
           "businessSn": "BSN2024041000001",
           "docSn": "CARD20240001",
           "productType": "IDCARD",
           "createTime": "2024-04-10T08:00:00Z",
           "dataInfo": {
             "dataPathType": "S3",
             "dataPackageUrl": "https://minio.example.com/...",
             "expireTime": "2024-04-10T10:00:00Z",
             "checksum": "sha256:...",
             "size": 2097152
           }
         }
       ]
     }
   }
   
   ◀──────────────────────────────────────────────

5. Download data package (based on dataPathType)
   
   Scenario A: dataPathType = S3
   GET https://minio.example.com/prepared-data/...?X-Amz-Algorithm=...
   
   Scenario B: dataPathType = FTP
   Use FTP client to download from ftp://ftp.example.com/...
   
   Scenario C: dataPathType = LOCAL
   Read local file: /data/prepared/BATCH20240410001_LINE001/BSN2024041000001.dat

6. Confirm batch start processing
   
   POST /open-api/mes/personalization/batches/confirm
   Authorization: Bearer {accessToken}
   X-Line-ID: LINE_001
   Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
   
   Request:
   {
     "batchSn": "BATCH20240410001_LINE001",
     "confirmTime": "2024-04-10T09:00:00Z",
     "operatorId": "OP001",
     "operatorName": "Operator Zhang San"
   }
   
   ──────────────────────────────────────────────▶
   
   Response:
   {
     "code": 0,
     "msg": "Batch confirmation successful",
     "data": {
       "batchSn": "BATCH20240410001_LINE001",
       "status": "PROCESSING",
       "confirmTime": "2024-04-10T09:00:00Z",
       "updateTime": "2024-04-10T09:00:01Z"
     }
   }
   
   ◀──────────────────────────────────────────────

7. Periodic heartbeat reporting during production (with current batch)
   
   POST /open-api/mes/personalization/heartbeat
   Authorization: Bearer {accessToken}
   X-Line-ID: LINE_001
   
   Request:
   {
     "lineId": "LINE_001",
     "deviceSn": "DEV_LINE001_001",
     "status": "RUNNING",
     "currentBatch": {
       "orderSn": "ORDER20240410001",
       "planSn": "PLAN20240410001",
       "batchSn": "BATCH20240410001_LINE001"
     },
     "progress": {
       "completed": 50,
       "total": 100,
       "percentage": 50.0
     },
     "reportTime": "2024-04-10T10:30:00Z"
   }
   
   ──────────────────────────────────────────────▶
   
   Response:
   {
     "code": 0,
     "data": {
       "hasNewTask": false,
       "nextHeartbeatInterval": 30
     }
   }
   
   ◀──────────────────────────────────────────────

8. Submit batch feedback after production completion
   
   POST /open-api/mes/personalization/batches/feedback
   Authorization: Bearer {accessToken}
   X-Line-ID: LINE_001
   Idempotency-Key: 550e8400-e29b-41d4-a716-446655440001
   
   Request:
   {
     "feedbackId": "FB20240410001",
     "batchSn": "BATCH20240410001_LINE001",
     "submitTime": "2024-04-10T12:00:00Z",
     "summary": {
       "totalCount": 100,
       "successCount": 98,
       "failCount": 2,
       "wasteCount": 4,
       "totalAttempts": 105
     },
     "results": [
       {
         "businessSn": "BSN2024041000001",
         "docSn": "CARD20240001",
         "status": "SUCCESS",
         "attemptCount": 1,
         "personalizeTime": "2024-04-10T10:05:00Z",
         "attempts": [
           {
             "attemptNo": 1,
             "status": "SUCCESS",
             "startTime": "2024-04-10T10:00:00Z",
             "endTime": "2024-04-10T10:05:00Z",
             "materials": [
               { "type": "CHIP", "id": "CHIP001" },
               { "type": "PVC", "id": "PVC001" },
               { "type": "RIBBON", "id": "RIBBON001" }
             ]
           }
         ]
       }
     ]
   }
   
   ──────────────────────────────────────────────▶
   
   Response:
   {
     "code": 0,
     "msg": "Feedback data received successfully",
     "data": {
       "feedbackId": "FB20240410001",
       "batchSn": "BATCH20240410001_LINE001",
       "accepted": true,
       "processingStatus": "QUEUED",
       "summary": {
         "totalCount": 100,
         "successCount": 98,
         "failCount": 2,
         "wasteCount": 4,
         "totalAttempts": 105
       }
     }
   }
   
   ◀──────────────────────────────────────────────
```

### 8.2 Important Notes

1. **Authentication and Token Management**
   - First call must perform authentication to obtain `accessToken`.
   - `accessToken` is valid for 2 hours by default (configurable). Refresh using `refresh_token` before expiration.
   - All business APIs must include `Authorization: Bearer {accessToken}` and `X-Line-ID: {lineId}` in request headers.
   - Authentication verifies the binding relationship between `lineId` and `deviceSn`.
   - Cache tokens locally to avoid frequent authentication.

2. **Heartbeat and Task Listening Mechanism**
   - Personalization system sends heartbeat requests periodically (recommended every 30 seconds).
   - Check the `hasNewTask` field in the heartbeat response.
   - If `hasNewTask = true`, immediately call **[3.1 Get Batch Task List](#31-get-batch-task-list)** API to fetch batches.
   - Avoid frequent polling of the batch list to reduce server load.

3. **Idempotency Design**
   - Modifying APIs (Confirm Batch Start, Submit Batch Feedback, etc.) must include the `Idempotency-Key` request header.
   - It is recommended to use a UUID for `Idempotency-Key`, with a unique value per request.
   - The server performs idempotency checks based on this key; duplicate requests will return the same result.

4. **Data Package Download Handling**
   - First, call **[3.2 Get Task Details List for a Batch](#32-get-task-details-list-for-a-batch)** to obtain `dataInfo` for each task.
   - Choose download method based on `dataPathType`:
     - `S3`: Use `dataPackageUrl` (pre-signed URL) to download from MinIO.
     - `FTP`: Use an FTP client to download from the FTP server.
     - `LOCAL`: Read the file from the local path.
   - Must verify checksum after download.
   - Pre-signed URLs are valid for 2 hours by default (configurable). Re-fetch task details if expired.

5. **Error Handling Strategy**
   - Network errors: Exponential backoff retry (1s → 2s → 4s → 8s), up to 5 times.
   - 401 errors: Token expired, use refresh_token or re-authenticate.
   - 429 errors: Rate limit triggered, reduce request frequency.
   - 5xx errors: Server exception, log and alert.
   - Business errors (code > 0): Handle according to the specific error code.

6. **Retry Mechanism Notes**
   - Multiple retries are allowed for the same `businessSn` after a production failure.
   - Update `attemptCount` for each retry.
   - Report `status: SUCCESS` and the actual `attemptCount` upon successful retry.
   - If max retries are exhausted, report `status: FAILED` and mark as final failure.

7. **Security Specifications**
   - All APIs must use HTTPS (except FTP).
   - Hardcoding `clientSecret` or `accessToken` is prohibited; use secure storage.
   - Temporary credentials like pre-signed URLs and FTP credentials must not be cached or distributed.
   - Discarded tokens should be cleaned up promptly.

---

**Document Version**: v1.0  
**Update Date**: 2024-04-13
```
