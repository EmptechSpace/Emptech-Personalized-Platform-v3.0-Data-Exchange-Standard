```markdown
# Production Management System API Documentation - Personalization System Integration

> **Document Version**: v1.0  
> **Target Audience**: Personalization System Developers  
> **Protocol**: HTTP RESTful + JSON  
> **Authentication**: Basic Auth + Bearer Token

---

## Table of Contents

1. [API Specification](#1-api-specification)
2. [Authentication API](#2-authentication-api)
3. [Task Pull API](#3-task-pull-api)
4. [Batch Confirmation API](#4-batch-confirmation-api)
5. [Batch Feedback API](#5-batch-feedback-api)
6. [Heartbeat and Status API](#6-heartbeat-and-status-api)
7. [Error Code Reference](#7-error-code-reference)
8. [Appendix](#8-appendix)

---

## 1. API Specification

### 1.1 API Path Convention

| API Type | Path Prefix | Example |
| --- | --- | --- |
| Authentication API | `/open-api/mes/auth/**` | `/open-api/mes/auth/token` |
| Business API | `/open-api/mes/personalization/**` | `/open-api/mes/personalization/batches/pending` |

### 1.2 Standard Request and Response Format

#### Standard Request Headers

| Header | Required | Description |
| --- | --- | --- |
| Authorization | Yes | `Bearer {accessToken}` |
| X-Line-ID | Yes | Unique production line identifier (lineId) |
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
| --- | --- | --- |
| code | int | Business status code, **0 indicates success**, non-zero indicates error |
| msg | string | Status or error description |
| data | object | Business data, null on failure |

**Error code notes:**

- `code = 0`: Business success
- `code > 0`: Business error, custom error code, with description in `msg`

### 1.3 Business Identifier Specification

| Identifier | Description | Example |
| --- | --- | --- |
| lineId | Unique production line ID | LINE_001 |
| deviceSn | Unique device ID | DEV_LINE001_001 |
| orderSn | Production order number | ORDER20240410001 |
| planSn | Production task plan number | PLAN20240410001 |
| batchSn | Batch number (production line level) | BATCH20240410001_LINE001 |
| businessSn | Unique production data identifier (per card) | BSN2024041000001 |
| docSn | Document number | B1231341 |

**Relationship description:**

- One production line (`lineId`) is bound to one device (`deviceSn`)
- One batch (`batchSn`) corresponds to one production line
- One batch contains multiple production data items (`businessSn`)

### 1.4 Data Package Type Specification

| dataPathType | Description | Handling Method |
| --- | --- | --- |
| S3 | MinIO object storage | Personalization system pulls data using pre-signed URL from MinIO |
| FTP | FTP file server | Personalization system downloads file from FTP server |
| LOCAL | Local file system | Personalization system reads file from local path |

---

## 2. Authentication API

### 2.1 Get Access Token

Obtain an access token using Basic authentication. The binding relationship between production line and device is verified during authentication.

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
| --- | --- | --- | --- |
| lineId | string | Yes | Unique production line identifier |
| deviceSn | string | Yes | Unique device identifier, must be bound to the production line |
| deviceInfo.deviceType | string | Yes | Device type |
| deviceInfo.softwareVersion | string | No | Software version (redundant) |
| deviceInfo.hardwareVersion | string | No | Hardware version (redundant) |

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

## 3. Task Pull API

### 3.1 Get Pending Batch List

Get the list of pending **batch tasks** for the current production line, sorted by priority and creation time.

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
| --- | --- | --- | --- | --- |
| limit | int | No | 10 | Number of batches to return, max 50 |

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
        "productName": "National ID Card",
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
        "productName": "National ID Card",
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
| --- | --- | --- |
| orderSn | string | Production order number |
| planSn | string | Production task plan number |
| batchSn | string | Batch number (production line level) |
| productType | string | Product type |
| productName | string | Product name |
| priority | int | Priority, lower number means higher priority |
| taskCount | int | Number of tasks in this batch |
| deadline | string | Deadline |
| createTime | string | Batch creation time |

---

### 3.2 Get Task Details List for a Batch

Get all pending production data details for a batch identified by `batchSn`. **Supports pagination**.

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
| --- | --- | --- | --- | --- |
| batchSn | string | Yes | - | Batch number |
| page | int | No | 1 | Page number |
| pageSize | int | No | 50 | Items per page, max 100 |

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
        "productName": "National ID Card",
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
        "productName": "National ID Card",
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
| --- | --- | --- |
| dataPathType | string | Data path type: S3/FTP/LOCAL |
| dataPackageUrl | string | Data package download URL (pre-signed URL or file path) |
| dataProxyPath | string | Data package proxy path, only LOCAL type; used when network is poor, data package pre-copied to factory |
| expireTime | string | Download URL expiration time (only for S3/FTP types) |
| checksum | string | Data package checksum (SHA-256) |
| size | long | Data package size (bytes) |

**dataPathType Description:**

| dataPathType | dataPackageUrl Format Example |
| --- | --- |
| S3 | `https://minio.example.com/...` (pre-signed URL) |
| FTP | `ftp://ftp.example.com/...` or `ftps://...` |
| LOCAL | `/data/prepared/BATCH20240410001_LINE001/BSN2024041000001.dat` |

---

## 4. Batch Confirmation API

### 4.1 Confirm Start of Batch Processing

Before processing a batch, the personalization system must call this API to confirm the start of the batch. **Confirmation is at batch level, not per businessSn.**

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
| --- | --- | --- | --- |
| batchSn | string | Yes | Batch number (production line level) |
| confirmTime | string | Yes | Confirmation time (ISO 8601 format) |
| expectedCompletionTime | string | No | Expected completion time |
| operatorId | string | No | Operator ID |
| operatorName | string | No | Operator name |

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
| --- | --- | --- |
| batchSn | string | Batch number |
| status | string | Batch status: PROCESSING - in progress |
| confirmTime | string | Confirmation time |
| updateTime | string | Update time |

---

## 5. Batch Feedback API

### 5.1 Submit Batch Personalization Results

After personalization is completed, submit the entire batch's results via this API. **Supports batch submission of the whole batch results, including detailed attempt records and material information.**

> **Note**: Considering batch data volume, this API supports multiple rounds of feedback. The client should implement its own sharding. The server will consider submission complete when it detects that the number of `businessSn` reaches the batch total.

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
          "errorMessage": "Print quality不合格",
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
| --- | --- | --- | --- |
| feedbackId | string | Yes | Unique feedback identifier for idempotency |
| batchSn | string | Yes | Batch number |
| submitTime | string | Yes | Submission time |
| summary | object | Yes | Batch summary information |
| summary.totalCount | int | Yes | Total number in batch |
| summary.successCount | int | Yes | Number of successes |
| summary.failCount | int | Yes | Number of failures |
| summary.wasteCount | int | Yes | Number of waste items |
| summary.totalAttempts | int | Yes | Total number of attempts |
| results | array | Yes | List of per-card personalization results |

**results Item Field Description:**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| businessSn | string | Yes | Unique production data identifier |
| docSn | string | Yes | Document number |
| status | string | Yes | Personalization result: SUCCESS/FAILED |
| attemptCount | int | Yes | Number of attempts |
| personalizeTime | string | Yes | Personalization completion time |
| attempts | array | Yes | Detailed information for each attempt |
| wasteSummary | object | No | Waste summary (when multiple attempts fail) |

**attempts Item Field Description:**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| attemptNo | int | Yes | Attempt sequence number |
| status | string | Yes | Attempt result: SUCCESS/FAILED |
| startTime | string | Yes | Attempt start time |
| endTime | string | Yes | Attempt end time |
| errorCode | string | No | Error code (required on failure) |
| errorMessage | string | No | Error description (optional on failure) |
| isWaste | boolean | No | Whether waste was generated |
| wasteType | string | No | Waste type |
| materials | array | Yes | Material usage information |

**materials Item Field Description:**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| type | string | Yes | Material type: CHIP/PVC/RIBBON |
| id | string | Yes | Unique material identifier |
| wasted | boolean | No | Whether scrapped (default false) |

---

## 6. Heartbeat and Status API

### 6.1 Report Heartbeat and Status

The personalization system periodically (recommended every 30 seconds) reports heartbeat and current status. The server returns an indicator whether there are new tasks.

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

**Key Heartbeat Response Fields:**

| Field | Type | Description |
| --- | --- | --- |
| hasNewTask | boolean | Whether there are new tasks pending |
| newTaskCount | int | Number of new tasks (optional) |
| nextHeartbeatInterval | int | Recommended next heartbeat interval (seconds) |

**Business Logic Description:**

Heartbeat handling logic for the personalization system:

1. Periodically (e.g., every 30 seconds) send a heartbeat request
2. Check the `hasNewTask` field in the response
3. If `hasNewTask = true`, immediately call the [3.1 Get Pending Batch List] API to fetch tasks
4. Process the retrieved task list

---

## 7. Error Code Reference

### 7.1 HTTP Status Codes

| Status Code | Description | Handling Suggestion |
| --- | --- | --- |
| 200 | Success | Process response normally |
| 400 | Bad request | Check request parameters against specification |
| 401 | Unauthorized | Token invalid or expired, re-authenticate |
| 403 | Forbidden | No permission to access the resource |
| 404 | Not found | Check resource ID |
| 409 | Conflict | Resource state conflict |
| 429 | Too many requests | Rate limited, reduce request frequency |
| 500 | Internal server error | Server exception, contact administrator |
| 503 | Service unavailable | Service temporarily unavailable, retry later |

### 7.2 Business Error Codes (code > 0)

| Error Code | Description | Handling Suggestion |
| --- | --- | --- |
| **Authentication** | | |
| 10001 | Authentication failed | Check clientId and clientSecret |
| 10002 | Token expired | Use refreshToken or re-authenticate |
| 10003 | Invalid token | Re-authenticate |
| 10004 | Production line not authorized | Contact administrator to configure line permissions |
| 10005 | Device does not match production line | Check that deviceSn is bound to lineId |
| **Task related** | | |
| 20001 | Batch not found | Check batchSn |
| 20002 | Batch already assigned | Batch occupied by another device, pull other batches |
| 20003 | Invalid batch state | Operation not allowed in current state |
| 20004 | Batch timeout | Batch exceeded max processing time, auto-released |
| 20005 | Batch does not belong to line | Batch not assigned to current production line |
| **Data file related** | | |
| 30001 | Data package not found | Check dataPackageUrl or contact administrator |
| 30002 | Data package download link expired | Re-fetch task details for new download link |
| 30003 | Data package checksum mismatch | Package may be corrupted, re-download or contact admin |
| 30004 | Unsupported data path type | Check dataPathType is S3/FTP/LOCAL |
| **Feedback related** | | |
| 40001 | Feedback already received | Already processed, do not resubmit |
| 40002 | Batch identifier not found | Check batchSn |
| 40003 | Invalid state transition | Cannot update to target state in current state |
| 40004 | Invalid production result data | Check productionResult field completeness |

### 7.3 Personalization Error Codes (used in errorCode field of feedback)

| Error Code | Category | Description | Retryable | Retry Strategy |
| --- | --- | --- | --- | --- |
| ERR_CHIP_WRITE_FAIL | Chip error | Chip write failed | Yes | Replace chip and retry |
| ERR_CHIP_READ_FAIL | Chip error | Chip read failed | Yes | Re-read |
| ERR_CHIP_VERIFY_FAIL | Chip error | Chip verification failed | No | Mark as failed |
| ERR_PRINTER_JAM | Print error | Printer paper jam | Yes | Clear jam and retry |
| ERR_PRINT_QUALITY | Print error | Print quality不合格 | No | Mark as failed |
| ERR_CARD_FEED_FAIL | Mechanical error | Card feed failed | Yes | Re-feed card |
| ERR_CARD_EJECT_FAIL | Mechanical error | Card eject failed | Yes | Manual intervention |
| ERR_DATA_CHECK_FAIL | Data error | Data validation failed | No | Mark as failed |
| ERR_DEVICE_OFFLINE | Device error | Device offline | Yes | Retry after device recovery |

---

## 8. Appendix

### 8.1 Complete Business Flow Sequence Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Business Flow                                   │
└─────────────────────────────────────────────────────────────────────────┘

1. Authenticate and get token

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

2. Periodically report heartbeat (recommended every 30 sec)

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

4. Get task details for a batch (get data package download info)

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

   Case A: dataPathType = S3
   GET https://minio.example.com/prepared-data/...?X-Amz-Algorithm=...

   Case B: dataPathType = FTP
   Use FTP client to download from ftp://ftp.example.com/...

   Case C: dataPathType = LOCAL
   Read local file: /data/prepared/BATCH20240410001_LINE001/BSN2024041000001.dat

6. Confirm batch start

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

7. Periodically report heartbeat during personalization (include current batch)

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

8. Submit batch feedback after personalization completion

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

   - The first call must authenticate to obtain an `accessToken`
   - The `accessToken` is valid for 2 hours by default (configurable). Use `refresh_token` before expiration
   - All business APIs must include `Authorization: Bearer {accessToken}` and `X-Line-ID: {lineId}` headers
   - Authentication verifies the binding between `lineId` and `deviceSn`
   - It is recommended to cache the token locally to avoid frequent authentication

2. **Heartbeat and Task Listening Mechanism**

   - The personalization system periodically (recommended every 30 seconds) sends heartbeat requests
   - Check the `hasNewTask` field in the heartbeat response
   - If `hasNewTask = true`, immediately call the [3.1 Get Pending Batch List] API to fetch batches
   - Avoid frequent polling of the batch list to reduce server load

3. **Idempotency Design**

   - Modification APIs (confirm batch start, submit batch feedback, etc.) must include the `Idempotency-Key` header
   - `Idempotency-Key` should be a UUID, different for each request
   - The server performs idempotency checks using this key; duplicate requests return the same result

4. **Data Package Download Handling**

   - First call the [3.2 Get Task Details List for a Batch] API to get `dataInfo` for each task
   - Choose download method based on `dataPathType`:
     - `S3`: Download from MinIO using `dataPackageUrl` (pre-signed URL)
     - `FTP`: Download from FTP server using an FTP client
     - `LOCAL`: Read file from local path
   - After download, must verify using `checksum`
   - Pre-signed URL is valid for 2 hours by default (configurable); re-fetch task details if expired

5. **Error Handling Strategy**

   - Network errors: Exponential backoff retry (1s → 2s → 4s → 8s), max 5 times
   - 401 errors: Token expired, use refresh_token or re-authenticate
   - 429 errors: Rate limited, reduce request frequency
   - 5xx errors: Server exception, log and alert
   - Business errors (code > 0): Handle according to error code

6. **Retry Mechanism Description**

   - The same `businessSn` can be retried multiple times after personalization failure
   - Update `attemptCount` for each retry
   - On success, report `status: SUCCESS` and the actual `attemptCount`
   - If still fails after reaching max retries, report `status: FAILED` and mark as final failure

7. **Security Specifications**

   - All APIs must use HTTPS (except FTP)
   - `clientSecret` and `accessToken` must not be hardcoded; use secure storage
   - Temporary credentials such as pre-signed URLs and FTP credentials must not be cached or shared
   - Discarded tokens should be cleaned up promptly

---

**Document Version**: v1.0  
**Last Updated**: 2024-04-13
```
