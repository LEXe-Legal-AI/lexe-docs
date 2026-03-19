---
owner: product-team
audience: public
source_of_truth: true
status: active
repo: lexe-docs
review_cadence: monthly
sensitivity: public
last_verified: 2026-03-19
---

# LEXe Legal AI - API Overview

## Authentication

All API requests require a Bearer token obtained through the LEXe OIDC authentication flow.

```
Authorization: Bearer <access_token>
```

Tokens are issued by the LEXe identity provider using standard OAuth 2.0 / OpenID Connect. Token lifetime and refresh behavior are configured per tenant.

## Base URL

```
https://api.lexe.pro
```

## Endpoints

### Chat Streaming

Start a legal research conversation with real-time streaming response.

```
POST /gateway/customer/chat/stream
Content-Type: application/json
Authorization: Bearer <token>
```

**Request Body**

```json
{
  "message": "Quali sono i termini di prescrizione per il risarcimento danni da sinistro stradale?",
  "conversation_id": "optional-existing-conversation-id",
  "persona_id": "optional-persona-id"
}
```

**Response**: Server-Sent Events (SSE) stream with the following event types:

| Event | Description |
|-------|-------------|
| `content` | Incremental text tokens of the response |
| `tool_call` | Notification that a legal tool is being invoked |
| `tool_result` | Summary of tool execution result |
| `citation` | Structured citation metadata |
| `confidence` | 3-band confidence assessment (VERIFIED / CAUTION / LOW_CONFIDENCE) |
| `done` | Stream completion with trace identifier |
| `error` | Error details if the request fails |

### Conversations

Manage conversation history.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/gateway/customer/conversations` | List conversations (paginated) |
| `GET` | `/gateway/customer/conversations/{id}` | Get conversation with messages |
| `DELETE` | `/gateway/customer/conversations/{id}` | Delete a conversation |

**Query Parameters** (list endpoint)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | integer | 1 | Page number |
| `limit` | integer | 20 | Items per page (max 100) |

### Memory

Store and retrieve contextual information across conversations.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/gateway/customer/memory/search` | Search memory by query |
| `POST` | `/gateway/customer/memory/store` | Store a memory entry |
| `GET` | `/gateway/customer/memory/{id}` | Retrieve a specific memory |

### Consent

Manage GDPR consent for data processing.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/gateway/customer/consent/status` | Check current consent status |
| `POST` | `/gateway/customer/consent/accept` | Record consent acceptance |

Consent must be granted before the chat endpoint will process requests. If consent has not been recorded, the chat endpoint returns `403 Forbidden` with a descriptive error.

### Evaluation

Submit quality ratings for assistant responses.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/gateway/customer/evaluate` | Submit a 1-5 star rating for a message |
| `GET` | `/gateway/customer/evaluate/{message_id}` | Retrieve existing rating |

**Rating Request Body**

```json
{
  "message_id": "message-uuid",
  "rating": 4,
  "comment": "Optional free-text feedback"
}
```

## Error Responses

All errors follow a consistent JSON structure:

```json
{
  "code": "ERROR_CODE",
  "message": "Human-readable description",
  "details": {}
}
```

**Common Error Codes**

| HTTP Status | Code | Meaning |
|-------------|------|---------|
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 403 | `HTTP_403` | Insufficient permissions or consent required |
| 404 | `NOT_FOUND` | Resource does not exist |
| 429 | `RATE_LIMITED` | Daily limit exceeded (conversations, tokens, or messages) |
| 500 | `INTERNAL_ERROR` | Server error; retry with exponential backoff |

## Rate Limits

Rate limits are determined by the tenant's plan. When a limit is reached, the API returns `429 Too Many Requests` with a `Retry-After` header indicating when the limit resets.

Limits may apply to:

- Daily conversation count
- Daily token consumption
- Concurrent requests

## SSE Client Notes

- Use an EventSource-compatible client or a streaming HTTP library
- Reconnection is handled by the client; the `conversation_id` in the initial request allows resumption
- Each SSE event is a JSON object prefixed by `data: `
- The stream terminates with a `done` event containing a trace hash for support reference

---

*For integration support, contact the LEXe technical team.*
