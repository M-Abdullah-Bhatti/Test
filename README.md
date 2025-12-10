# Database Connector API Integration Guide

## Overview

This update adds **Database Connector** functionality allowing users to connect external databases and query them using natural language through the chat interface.



---

## Authentication

All connector endpoints require **HMAC-signed requests**.

### Required Headers

| Header          | Description                          |
|:----------------|:-------------------------------------|
| `Authorization` | `Bearer <api_key>`                   |
| `X-Timestamp`   | Unix timestamp in seconds            |
| `X-Signature`   | HMAC-SHA256 signature (see below)    |

### Signature Computation

```
signature = HMAC-SHA256(
    key = api_key_secret,
    message = timestamp + method + path + body
)
```

- `timestamp`: Same value as `X-Timestamp` header
- `method`: HTTP method in uppercase (GET, POST, etc.)
- `path`: Full path including query string (e.g., `/connectors?project_id=abc123`)
- `body`: Request body as string (empty string for GET requests)

---

## Endpoints

### 1. List Available Connector Types

Get connector types available for creation. Use this to populate UI dropdowns.

```
GET /connectors/types?project_id={project_id}
```

**Response:**
```json
[
  {
    "agent_type": "postgresql",
    "agent_class": "database_connector",
    "display_name": "PostgreSQL",
    "description": "Connect to PostgreSQL databases",
    "credential_fields": [
      {"name": "host", "type": "text", "required": true},
      {"name": "port", "type": "number", "required": true, "default": 5432},
      {"name": "database", "type": "text", "required": true},
      {"name": "user", "type": "text", "required": true},
      {"name": "password", "type": "password", "required": true},
      {"name": "schema", "type": "text", "required": false, "default": "public"}
    ],
    "example_queries": [
      "How many records are in the users table?",
      "Show top 10 customers by revenue"
    ]
  }
]
```

---

### 2. List Project Connectors

Get all connectors configured for a project.

```
GET /connectors?project_id={project_id}
```

**Response:**
```json
{
  "connectors": [
    {
      "agent_id": "550e8400-e29b-41d4-a716-446655440000",
      "agent_class": "database_connector",
      "agent_type": "postgresql",
      "name": "Production DB",
      "description": "Main analytics database",
      "enabled": true,
      "is_default": false,
      "initialization_status": "done",
      "created_at": "2025-12-10T10:00:00Z",
      "updated_at": "2025-12-10T10:05:00Z"
    }
  ],
  "total": 1
}
```

---

### 3. Create Connector

Create a new database connector for a project.

```
POST /connectors?project_id={project_id}
```

**Request Body:**
```json
{
  "agent_type": "postgresql",
  "name": "My Database",
  "description": "Production analytics database",
  "credentials": {
    "host": "db.example.com",
    "port": 5432,
    "database": "analytics",
    "user": "readonly_user",
    "password": "secret123",
    "schema": "public"
  },
  "settings": {}
}
```

**Response:** `201 Created`
```json
{
  "agent_id": "550e8400-e29b-41d4-a716-446655440000",
  "agent_class": "database_connector",
  "agent_type": "postgresql",
  "name": "My Database",
  "description": "Production analytics database",
  "enabled": true,
  "is_default": false,
  "initialization_status": "pending",
  "created_at": "2025-12-10T10:00:00Z",
  "updated_at": "2025-12-10T10:00:00Z"
}
```

**Note:** After creation, the backend asynchronously:
1. Tests database connection
2. Extracts schema (tables, columns, relationships)
3. Indexes schema for natural language queries

Poll the status endpoint to check when initialization completes.

---

### 4. Get Connector Status

Check initialization status of a connector.

```
GET /connectors/{connector_id}/status?project_id={project_id}
```

**Response:**
```json
{
  "connector_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "My Database",
  "agent_type": "postgresql",
  "initialization_status": "done",
  "initialization_error": null,
  "initialized_at": "2025-12-10T10:05:00Z",
  "schema_summary": {
    "table_count": 25,
    "column_count": 180,
    "database_name": "analytics",
    "schema_name": "public",
    "model_names": ["users", "orders", "products", "..."]
  }
}
```

**Status Values:**

| Status       | Meaning                                      |
|:-------------|:---------------------------------------------|
| `pending`    | Queued, initialization not yet started       |
| `processing` | Currently connecting and extracting schema   |
| `done`       | Ready for queries                            |
| `failed`     | Error occurred, check `initialization_error` |

---

### 5. Test Connector

Test connector connectivity and health.

```
POST /connectors/{connector_id}/test?project_id={project_id}
```

**Response:**
```json
{
  "agent_id": "550e8400-e29b-41d4-a716-446655440000",
  "healthy": true,
  "message": "Connection successful",
  "response_time_ms": 45
}
```

---

### 6. Update Connector

Update connector configuration. All fields are optional.

```
PUT /connectors/{connector_id}?project_id={project_id}
```

**Request Body:**
```json
{
  "name": "Updated Name",
  "description": "Updated description",
  "enabled": false,
  "credentials": {
    "host": "new-db.example.com",
    "port": 5432,
    "database": "analytics",
    "user": "new_user",
    "password": "new_password"
  },
  "settings": {}
}
```

**Response:** Updated connector object

---

### 7. Delete Connector

Delete a connector (soft delete).

```
DELETE /connectors/{connector_id}?project_id={project_id}
```

**Response:** `204 No Content`

**Note:** Default connectors (Document Search) cannot be deleted.

---

## Supported Database Types

| Database   | `agent_type` Value | Required Credentials                       |
|:-----------|:-------------------|:-------------------------------------------|
| PostgreSQL | `postgresql`       | host, port, database, user, password       |

### PostgreSQL Credentials Schema

```json
{
  "host": "string (required)",
  "port": "integer (required, default: 5432)",
  "database": "string (required)",
  "user": "string (required)",
  "password": "string (required)",
  "schema": "string (optional, default: public)"
}
```

---

## Using Connectors in Chat

Once a connector has `initialization_status: "done"`, users can ask database questions through the existing chat endpoint. The system automatically routes queries to the appropriate data source.

```
POST /chat/message?project_id={project_id}
```

**Request:**
```json
{
  "message": "How many customers signed up last month?",
  "conversation_id": "optional-conversation-uuid"
}
```

**Response:**
```json
{
  "response": "Based on the database, 1,247 customers signed up last month.",
  "conversation_id": "conversation-uuid",
  "message_id": "message-uuid",
  "source": "database",
  "metadata": {
    "sql": "SELECT COUNT(*) FROM customers WHERE created_at >= '2025-11-01'",
    "tables_used": ["customers"],
    "row_count": 1
  }
}
```

### Automatic Query Routing

The system automatically determines which data source to query based on the message content.

**Routed to Database:**
- "how many", "count", "total", "sum", "average"
- "list", "show", "get", "find", "select"
- "sales", "revenue", "orders", "customers", "users"
- "last month", "this week", "yesterday", "today"
- "top", "bottom", "highest", "lowest"

**Routed to Document Search:**
- "what is", "explain", "describe", "define"
- "document", "file", "pdf", "report"
- "according to", "based on", "mentioned in"

No special flags or parameters needed - routing is automatic.

---

## Integration Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  1. GET /connectors/types                                       │
│     → Get available connector types for UI dropdown             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. POST /connectors                                            │
│     → Create connector with database credentials                │
│     → Returns connector with status: "pending"                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. GET /connectors/{id}/status (poll)                          │
│     → Poll until initialization_status = "done"                 │
│     → Typical time: 10-60 seconds depending on schema size      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. POST /chat/message                                          │
│     → Ask natural language questions                            │
│     → System auto-routes to database or documents               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Error Responses

| HTTP Code | Meaning                                                |
|:----------|:-------------------------------------------------------|
| 400       | Invalid request (bad project_id format, missing fields)|
| 401       | Invalid or expired API key                             |
| 403       | API key lacks project access, or invalid HMAC signature|
| 404       | Connector not found                                    |
| 429       | Rate limit exceeded (10 requests/second)               |

### Error Response Format

```json
{
  "detail": "Error message describing the issue"
}
```

### Common Errors

**Invalid HMAC Signature (403):**
```json
{
  "detail": "Invalid request signature"
}
```
Check that signature computation includes the full path with query string.

**Connector Not Ready:**
```json
{
  "detail": "Connector 'My Database' is currently initializing. Please wait a moment and try again."
}
```
Poll the status endpoint and wait for `initialization_status: "done"`.

**Initialization Failed:**
```json
{
  "detail": "Connector 'My Database' initialization failed: Connection refused. Please delete and recreate the connector."
}
```
Check credentials and network connectivity, then recreate the connector.

---

## Rate Limits

| Endpoint Type | Limit           |
|:--------------|:----------------|
| Connectors    | 10 requests/sec |
| Chat          | 10 requests/sec |
| Auth          | 10 requests/sec |

Rate limit headers are included in all responses:
- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`
- `X-RateLimit-Reset`

---

## Best Practices

1. **Poll status after creation**: Don't assume the connector is ready immediately. Poll `/connectors/{id}/status` until `initialization_status` is `done`.

2. **Use read-only credentials**: For security, provide database credentials with read-only permissions (SELECT only).

3. **Handle initialization failures**: If status is `failed`, display the `initialization_error` to the user and prompt them to check credentials.

4. **Cache connector types**: The `/connectors/types` response rarely changes. Cache it to reduce API calls.

5. **Implement retry logic**: For chat queries, implement exponential backoff if you receive 429 or 5xx errors.
