# ALHAI API Developer Guide

## Document Upload & Management API

**Base URL:** `http://your-server:8412`

This guide explains how to authenticate, set up organizations/projects, manage users, and upload documents using the ALHAI API.

---

## Table of Contents

1. [Authentication Overview](#1-authentication-overview)
2. [Step 1: User Registration & Login](#2-step-1-user-registration--login)
3. [Step 2: Create Organization & Project](#3-step-2-create-organization--project)
4. [Step 3: Add Users to Projects](#4-step-3-add-users-to-projects)
5. [Step 4: Generate API Key with Secret](#5-step-4-generate-api-key-with-secret)
6. [Step 5: HMAC Request Signing](#6-step-5-hmac-request-signing)
7. [Step 6: Upload Documents](#7-step-6-upload-documents)
8. [Step 7: Monitor Document Status](#8-step-7-monitor-document-status)
9. [Complete Flow Diagram](#9-complete-flow-diagram)
10. [Error Handling](#10-error-handling)

---

## 1. Authentication Overview

ALHAI uses a **dual authentication system**:

| Auth Type | Used For | How It Works |
|-----------|----------|--------------|
| **JWT Tokens** | User sessions, admin tasks | Login with username/password, get access token |
| **API Keys + HMAC** | Document uploads, data access | Generate API key, sign every request with HMAC-SHA256 |

### When to Use What

```
JWT Token (Bearer Token):
├── User registration/login
├── Create organizations
├── Create projects
├── Add/remove users
├── Generate API keys
└── Session management

API Key + HMAC Signature:
├── Upload documents
├── List documents
├── Delete documents
├── Chat endpoints
└── Data retrieval
```

---

## 2. Step 1: User Registration & Login

### 2.1 Register a New User

```
POST /auth/signup
Content-Type: application/json

{
    "username": "john_doe",
    "email": "john@example.com",
    "password": "SecurePass123!",
    "full_name": "John Doe"
}
```

**Response (200 OK):**
```json
{
    "id": "6926fad6c55f4c105843590e",
    "username": "john_doe",
    "email": "john@example.com",
    "full_name": "John Doe",
    "is_active": true,
    "created_at": "2025-11-26T13:04:22.000Z"
}
```

### 2.2 Login (Get JWT Token)

**Option A: Form Data (OAuth2 standard)**
```
POST /auth/login
Content-Type: application/x-www-form-urlencoded

username=john_doe&password=SecurePass123!
```

**Option B: JSON Body**
```
POST /auth/login-json
Content-Type: application/json

{
    "username": "john_doe",
    "password": "SecurePass123!"
}
```

**Response (200 OK):**
```json
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "V0iNTMioVe_LwLi60dxjARfRut5LFR_4...",
    "token_type": "bearer",
    "expires_in": 1800
}
```

**Important:**
- `access_token` expires in 30 minutes (1800 seconds)
- `refresh_token` is used to get new access tokens
- Store both tokens securely

### 2.3 Using the JWT Token

For all JWT-protected endpoints, include the token in the Authorization header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 2.4 Refresh Token (When Access Token Expires)

```
POST /auth/refresh
Content-Type: application/json

{
    "refresh_token": "V0iNTMioVe_LwLi60dxjARfRut5LFR_4..."
}
```

**Response:**
```json
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "XfK5FmxSZ9W7X53VKM6h...",
    "token_type": "bearer",
    "expires_in": 1800
}
```

---

## 3. Step 2: Create Organization & Project

### 3.1 Create an Organization

Organizations are the top-level container. The user who creates it becomes the **owner**.

```
POST /organizations/create
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "Acme Corporation",
    "description": "Our company workspace"
}
```

**Response (200 OK):**
```json
{
    "id": "6926fad8c55f4c1058435911",
    "name": "Acme Corporation",
    "description": "Our company workspace",
    "message": "Organization created successfully"
}
```

**Save the `id`** - you'll need it to create projects.

### 3.2 Create a Project

Projects belong to organizations. Only the **organization owner** can create projects.

```
POST /organizations/{org_id}/projects/create
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "Research Documents",
    "description": "All research papers and reports",
    "organization_id": "6926fad8c55f4c1058435911"
}
```

**Response (200 OK):**
```json
{
    "id": "6926fad8c55f4c1058435912",
    "name": "Research Documents",
    "description": "All research papers and reports",
    "organization_id": "6926fad8c55f4c1058435911",
    "message": "Project created successfully"
}
```

**Save the project `id`** - you'll need it for API key generation and document uploads.

### 3.3 List Your Organizations

```
GET /organizations/list
Authorization: Bearer {access_token}
```

**Response:**
```json
[
    {
        "id": "6926fad8c55f4c1058435911",
        "name": "Acme Corporation",
        "description": "Our company workspace",
        "is_owner": true
    }
]
```

### 3.4 List Projects in an Organization

```
GET /organizations/{org_id}/projects
Authorization: Bearer {access_token}
```

**Response:**
```json
[
    {
        "id": "6926fad8c55f4c1058435912",
        "name": "Research Documents",
        "description": "All research papers and reports",
        "role": "admin",
        "created_at": "2025-11-26T13:04:24.000Z"
    }
]
```

---

## 4. Step 3: Add Users to Projects

### User Roles

| Role | Can Upload Documents | Can Add/Remove Users | Can Delete Any Document |
|------|---------------------|---------------------|------------------------|
| `admin` | Yes | Yes | Yes (project scope) |
| `user` | Yes | No | Only own documents |

### 4.1 Add a User to a Project

**Only project admins or organization owners can add users.**

First, the user must be registered in the system. Then add them by email:

```
POST /organizations/{org_id}/projects/{project_id}/add-user
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "user_email": "jane@example.com",
    "role": "user"
}
```

**Response (200 OK):**
```json
{
    "message": "User jane_doe added to project successfully",
    "role": "user"
}
```

### 4.2 List Users in a Project

```
GET /organizations/{org_id}/projects/{project_id}/users
Authorization: Bearer {access_token}
```

**Response:**
```json
[
    {
        "id": "6926fad6c55f4c105843590e",
        "username": "john_doe",
        "email": "john@example.com",
        "full_name": "John Doe",
        "role": "admin",
        "added_at": "2025-11-26T13:04:24.000Z"
    },
    {
        "id": "6926fad9c55f4c1058435918",
        "username": "jane_doe",
        "email": "jane@example.com",
        "full_name": "Jane Doe",
        "role": "user",
        "added_at": "2025-11-26T13:05:00.000Z"
    }
]
```

### 4.3 Remove a User from a Project

```
DELETE /organizations/{org_id}/projects/{project_id}/remove-user/{user_email}
Authorization: Bearer {access_token}
```

**Response:**
```json
{
    "message": "User jane_doe removed from project"
}
```

---

## 5. Step 4: Generate API Key with Secret

API keys are required for document operations. Each key comes with a **secret** for HMAC signing.

### 5.1 Generate API Key

```
POST /api-keys/generate
Authorization: Bearer {access_token}
Content-Type: application/json

{
    "name": "Document Upload Key",
    "description": "API key for uploading research documents",
    "project_ids": ["6926fad8c55f4c1058435912"]
}
```

**Response (200 OK):**
```json
{
    "id": "6926fad9c55f4c105843591a",
    "key": "sk_1g1mAX4Ph3TfrayzxNAHjlC1n2K3L6VU",
    "secret": "ss_RVabgngiCoTGVsgpvVvps2kRRYsIVcQG",
    "name": "Document Upload Key",
    "description": "API key for uploading research documents",
    "created_at": "2025-11-26T13:04:24.718Z",
    "is_active": true
}
```

**CRITICAL:**
- The `key` and `secret` are **only shown once** at creation time
- Store them securely immediately
- The `secret` is required for HMAC signing (see next section)

### 5.2 List Your API Keys

```
GET /api-keys/list
Authorization: Bearer {access_token}
```

**Response:**
```json
[
    {
        "id": "6926fad9c55f4c105843591a",
        "key": "sk_1g1mAX4P****",
        "name": "Document Upload Key",
        "description": "API key for uploading research documents",
        "created_at": "2025-11-26T13:04:24.718Z",
        "is_active": true
    }
]
```

Note: Full key is never shown again after creation.

### 5.3 Revoke an API Key

```
DELETE /api-keys/{key_id}
Authorization: Bearer {access_token}
```

### 5.4 Reactivate a Revoked Key

```
POST /api-keys/{key_id}/reactivate
Authorization: Bearer {access_token}
```

---

## 6. Step 5: HMAC Request Signing

All document/data endpoints require **HMAC-SHA256 signature** for security.

### Why HMAC?

- Ensures request wasn't tampered with
- Proves you have the secret key
- Prevents replay attacks (via timestamp + nonce)

### Required Headers for Signed Requests

| Header | Description | Example |
|--------|-------------|---------|
| `Authorization` | API key | `Bearer sk_1g1mAX4Ph3Tfrayz...` |
| `X-Signature` | HMAC-SHA256 signature | `0869c2ce341cdf1b35b0e4ff...` |
| `X-Timestamp` | Unix timestamp (seconds) | `1764162265` |
| `X-Nonce` | Random unique string | `1BPTPH6dCoppa1x-4UuF...` |

### How to Generate the Signature

**Step 1: Create the Canonical Request String**

```
{METHOD}\n{PATH}\n{BODY}\n{TIMESTAMP}\n{NONCE}
```

Example for a GET request:
```
GET
/ingestion/documents/6926fad8c55f4c1058435912

1764162265
abc123random456
```

Example for a POST request with body:
```
POST
/ingestion/upload
project_id=123&organization_id=456&file=...
1764162265
xyz789unique012
```

**Step 2: Generate HMAC-SHA256**

```
signature = HMAC-SHA256(secret_key, canonical_request)
```

**Pseudocode Example:**

```
function signRequest(method, path, body, apiKeySecret):
    timestamp = currentUnixTimestamp()           // e.g., 1764162265
    nonce = generateRandomString(24)             // e.g., "abc123xyz789..."

    canonicalRequest = method + "\n"
                     + path + "\n"
                     + body + "\n"
                     + timestamp + "\n"
                     + nonce

    signature = hmacSha256(apiKeySecret, canonicalRequest)

    return {
        "X-Signature": signature,
        "X-Timestamp": timestamp,
        "X-Nonce": nonce
    }
```

**Important Rules:**
- Timestamp must be within **5 minutes** of server time
- Each nonce can only be used **once**
- For GET requests, body is empty string
- For POST with form data, use the raw form body
- For POST with JSON, use the JSON string

---

## 7. Step 6: Upload Documents

### 7.1 Upload a Document

```
POST /ingestion/upload
Authorization: Bearer sk_1g1mAX4Ph3TfrayzxNAHjlC1n2K3L6VU
X-Signature: {hmac_signature}
X-Timestamp: {unix_timestamp}
X-Nonce: {random_nonce}
Content-Type: multipart/form-data

Form Fields:
- project_id: "6926fad8c55f4c1058435912"
- organization_id: "6926fad8c55f4c1058435911"
- file: [binary file data]
- metadata: {"author": "John", "category": "research"} (optional, JSON string)
```

**Response (200 OK):**
```json
{
    "status": "Your document research_paper.pdf is uploaded",
    "document_id": "7d70fad2-2bf7-4728-8be2-aaea81d0eb32",
    "filename": "research_paper.pdf",
    "processing_status": "pending"
}
```

**Save the `document_id`** - use it to check processing status.

### 7.2 Supported File Types

| Type | Content-Type | Notes |
|------|--------------|-------|
| PDF | `application/pdf` | Processed with InternVL vision model |
| Text | `text/plain` | Direct text extraction |
| Other | various | Treated as text |

### 7.3 What Happens After Upload

```
Upload → pending → processing → done
                              ↘ failed (check processing_error)
```

1. Document is saved to database
2. Async task is queued for processing
3. PDF is parsed (text + images + tables)
4. Text is chunked for retrieval
5. Embeddings are generated
6. Chunks stored in vector database
7. Status updated to "done"

---

## 8. Step 7: Monitor Document Status

### 8.1 Check Document Status

```
GET /ingestion/status/{document_id}
Authorization: Bearer sk_1g1mAX4Ph3TfrayzxNAHjlC1n2K3L6VU
X-Signature: {hmac_signature}
X-Timestamp: {unix_timestamp}
X-Nonce: {random_nonce}
```

**Response:**
```json
{
    "document_id": "7d70fad2-2bf7-4728-8be2-aaea81d0eb32",
    "filename": "research_paper.pdf",
    "content_type": "pdf",
    "processing_status": "done",
    "chromadb_stored": true,
    "chunk_count": 42,
    "uploaded_at": "2025-11-26T13:04:25.000Z",
    "processed_at": "2025-11-26T13:06:30.000Z",
    "processing_error": null
}
```

### Processing Status Values

| Status | Meaning |
|--------|---------|
| `pending` | Uploaded, waiting in queue |
| `processing` | Currently being processed |
| `done` | Successfully processed and indexed |
| `failed` | Processing failed (see `processing_error`) |

### 8.2 List Documents in Project

```
GET /ingestion/documents/{project_id}?limit=50&skip=0
Authorization: Bearer sk_1g1mAX4Ph3TfrayzxNAHjlC1n2K3L6VU
X-Signature: {hmac_signature}
X-Timestamp: {unix_timestamp}
X-Nonce: {random_nonce}
```

**Response:**
```json
{
    "documents": [
        {
            "id": "6926fad9c55f4c105843591c",
            "filename": "research_paper.pdf",
            "content_type": "pdf",
            "file_size": 377283,
            "uploaded_at": "2025-11-26T13:04:25.000Z",
            "processed": true,
            "processing_status": "done"
        }
    ],
    "total": 1,
    "limit": 50,
    "skip": 0
}
```

### 8.3 List Pending Documents

```
GET /ingestion/pending?project_id={project_id}&limit=100
Authorization: Bearer {api_key}
X-Signature: {hmac_signature}
X-Timestamp: {unix_timestamp}
X-Nonce: {random_nonce}
```

### 8.4 Reprocess a Failed Document

```
POST /ingestion/reprocess/{document_id}
Authorization: Bearer {api_key}
X-Signature: {hmac_signature}
X-Timestamp: {unix_timestamp}
X-Nonce: {random_nonce}
```

**Response:**
```json
{
    "message": "Document queued for reprocessing",
    "document_id": "7d70fad2-2bf7-4728-8be2-aaea81d0eb32",
    "previous_status": "failed",
    "new_status": "pending"
}
```

### 8.5 Delete a Document

**Note:** Only the document owner can delete their documents.

```
DELETE /ingestion/documents/{document_id}
Authorization: Bearer {api_key}
X-Signature: {hmac_signature}
X-Timestamp: {unix_timestamp}
X-Nonce: {random_nonce}
```

**Response:**
```json
{
    "message": "Document deleted successfully"
}
```

---

## 9. Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         COMPLETE API FLOW                                │
└─────────────────────────────────────────────────────────────────────────┘

                              SETUP PHASE (One-time)
                              ─────────────────────

    ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
    │   Register   │ ──────► │    Login     │ ──────► │  Get JWT     │
    │    User      │         │              │         │   Token      │
    └──────────────┘         └──────────────┘         └──────┬───────┘
                                                              │
                    ┌─────────────────────────────────────────┘
                    │
                    ▼
    ┌──────────────────────────────────────────────────────────────────┐
    │                    WITH JWT TOKEN (Authorization: Bearer)         │
    └──────────────────────────────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────────┐
    │ Create │  │ Create │  │ Add Users  │
    │  Org   │  │Project │  │ to Project │
    └────┬───┘  └────┬───┘  └────────────┘
         │           │
         │     ┌─────┴─────┐
         │     ▼           │
         │  ┌────────────┐ │
         │  │ Generate   │ │
         │  │ API Key +  │◄┘
         │  │  Secret    │
         │  └─────┬──────┘
         │        │
         │        │  SAVE KEY & SECRET!
         │        │  (shown only once)
         │        │
         └────────┼────────────────────────────────────────────────────┐
                  │                                                     │
                  ▼                                                     │
                              OPERATION PHASE (Repeatable)              │
                              ────────────────────────────              │
                                                                        │
    ┌──────────────────────────────────────────────────────────────────┐│
    │            WITH API KEY + HMAC SIGNATURE                          ││
    │   Headers: Authorization, X-Signature, X-Timestamp, X-Nonce       ││
    └──────────────────────────────────────────────────────────────────┘│
                  │                                                     │
        ┌─────────┼─────────┬─────────────┐                            │
        ▼         ▼         ▼             ▼                            │
    ┌────────┐┌────────┐┌────────┐  ┌──────────┐                       │
    │Upload  ││ List   ││ Check  │  │  Delete  │                       │
    │Document││Documents││ Status │  │ Document │                       │
    └────┬───┘└────────┘└────────┘  └──────────┘                       │
         │                                                              │
         ▼                                                              │
    ┌──────────┐    ┌──────────┐    ┌──────────┐                       │
    │ pending  │───►│processing│───►│   done   │                       │
    └──────────┘    └──────────┘    └──────────┘                       │
                          │                                             │
                          ▼                                             │
                    ┌──────────┐                                        │
                    │  failed  │─── Reprocess ─────────────────────────┘
                    └──────────┘
```

---

## 10. Error Handling

### Common HTTP Status Codes

| Code | Meaning | What to Do |
|------|---------|------------|
| 200 | Success | Request completed |
| 400 | Bad Request | Check request format/parameters |
| 401 | Unauthorized | Token expired or invalid |
| 403 | Forbidden | No permission for this action |
| 404 | Not Found | Resource doesn't exist |
| 429 | Rate Limited | Too many requests, wait and retry |
| 500 | Server Error | Retry later, contact support |

### Common Error Responses

**Invalid/Expired JWT Token:**
```json
{
    "detail": "Could not validate credentials"
}
```
→ Solution: Refresh token or login again

**Invalid HMAC Signature:**
```json
{
    "detail": "Invalid request signature"
}
```
→ Solution: Check signature generation, timestamp sync

**Timestamp Too Old:**
```json
{
    "detail": "Request timestamp too old"
}
```
→ Solution: Sync client time, use current timestamp

**No Project Access:**
```json
{
    "detail": "API key does not have access to this project"
}
```
→ Solution: Verify API key was created with correct project_ids

**Rate Limited:**
```json
{
    "detail": "Rate limit exceeded"
}
```
→ Solution: Wait for `X-RateLimit-Reset` header time

### Rate Limits

| Endpoint Type | Limit |
|---------------|-------|
| Authentication | 5 requests/minute |
| API Operations | 1000 requests/hour |
| General | Varies by endpoint |

---

## Quick Reference Card

### JWT Token Endpoints (use `Authorization: Bearer {jwt_token}`)

| Action | Method | Endpoint |
|--------|--------|----------|
| Register | POST | `/auth/signup` |
| Login | POST | `/auth/login` or `/auth/login-json` |
| Get Profile | GET | `/auth/me` |
| Refresh Token | POST | `/auth/refresh` |
| Logout | POST | `/auth/logout` |
| Create Org | POST | `/organizations/create` |
| List Orgs | GET | `/organizations/list` |
| Create Project | POST | `/organizations/{org_id}/projects/create` |
| List Projects | GET | `/organizations/{org_id}/projects` |
| Add User | POST | `/organizations/{org_id}/projects/{project_id}/add-user` |
| Remove User | DELETE | `/organizations/{org_id}/projects/{project_id}/remove-user/{email}` |
| List Users | GET | `/organizations/{org_id}/projects/{project_id}/users` |
| Generate API Key | POST | `/api-keys/generate` |
| List API Keys | GET | `/api-keys/list` |
| Revoke API Key | DELETE | `/api-keys/{key_id}` |

### API Key + HMAC Endpoints (use all 4 headers)

| Action | Method | Endpoint |
|--------|--------|----------|
| Upload Document | POST | `/ingestion/upload` |
| List Documents | GET | `/ingestion/documents/{project_id}` |
| Get Status | GET | `/ingestion/status/{document_id}` |
| Delete Document | DELETE | `/ingestion/documents/{document_id}` |
| List Pending | GET | `/ingestion/pending` |
| Reprocess | POST | `/ingestion/reprocess/{document_id}` |

---

## Need Help?

- API Documentation: `http://your-server:8412/docs` (Swagger UI)
- Alternative Docs: `http://your-server:8412/redoc` (ReDoc)
- Health Check: `GET http://your-server:8412/health`
