---
name: inboxparse
description: >
  Work with the InboxParse Email-to-LLM API. Use when the user mentions
  "InboxParse", "email API", "parse emails", "email-to-LLM", "email parsing",
  or wants to integrate email data into AI workflows. Covers authentication,
  listing/searching/sending emails, threads, labels, mailboxes, webhooks,
  and usage tracking.
metadata:
  version: "1.0.0"
  author: "InboxParse"
  homepage: "https://inboxparse.com"
license: MIT
---

# InboxParse API Skill

Turn emails into structured, AI-ready data via a clean REST API.

## What This Skill Does

Guides agents through the InboxParse V1 API to:
- Read, search, and retrieve emails and threads
- Send and reply to emails
- Manage mailboxes, labels, and webhooks
- Track API usage and quotas

## When to Use

- User wants to integrate email data into an application
- User asks about InboxParse API endpoints or authentication
- User wants to set up email webhooks or labels
- User needs to search emails or retrieve thread data
- User is building an AI agent that processes email

## When NOT to Use

- General email client questions (Outlook, Gmail UI)
- SMTP server configuration unrelated to InboxParse
- Questions about other email APIs (SendGrid, Postmark, etc.)

---

## Authentication

**Base URL**: `https://inboxparse.com/api/v1`

**API Key** (required for all endpoints):

```bash
# Preferred
Authorization: Bearer ip_<your_key>

# Alternative
X-API-Key: ip_<your_key>
```

- Keys use the `ip_` prefix followed by 64 hex characters
- **Member keys**: read-only (GET endpoints only)
- **Admin keys**: full access (GET + POST/PATCH/DELETE)
- Get keys from the InboxParse dashboard at `/console/api-keys`

---

## Response Format

**Single item**: `{ "data": { ... } }`

**List**: `{ "data": [...], "count": 5 }`

**Paginated list**:
```json
{
  "data": [...],
  "pagination": { "next_cursor": "abc123", "has_more": true }
}
```

**Error**:
```json
{
  "error": { "message": "Human-readable message", "code": "error_code" }
}
```

---

## Endpoints Quick Reference

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/emails` | member | List emails (paginated) |
| GET | `/emails/:id` | member | Get single email |
| POST | `/emails/send` | admin | Send new email |
| POST | `/emails/reply` | admin | Reply to email |
| GET | `/threads` | member | List threads |
| GET | `/threads/:id` | member | Get thread with messages |
| POST | `/search` | member | Search emails |
| GET | `/mailboxes` | member | List mailboxes |
| POST | `/mailboxes` | admin | Create IMAP mailbox |
| DELETE | `/mailboxes/:id` | admin | Delete mailbox |
| GET | `/labels` | member | List labels |
| POST | `/labels` | admin | Create label |
| PATCH | `/labels/:id` | admin | Update label |
| DELETE | `/labels/:id` | admin | Delete label |
| GET | `/webhooks` | member | List webhooks |
| POST | `/webhooks` | admin | Create webhook |
| PATCH | `/webhooks/:id` | admin | Update webhook |
| DELETE | `/webhooks/:id` | admin | Delete webhook |
| POST | `/webhooks/:id/test` | admin | Test webhook |
| GET | `/webhooks/:id/deliveries` | member | Delivery history |
| GET | `/usage` | member | Usage stats |

---

## Common Workflows

### List Recent Emails

```bash
curl -H "Authorization: Bearer ip_<key>" \
  "https://inboxparse.com/api/v1/emails?limit=10&format=markdown"
```

Query params: `limit` (max 100), `cursor`, `date_from`, `date_to`, `mailbox_id`, `direction` (inbound/outbound), `format` (markdown/full/raw).

### Get Email with AI Analysis

```bash
curl -H "Authorization: Bearer ip_<key>" \
  "https://inboxparse.com/api/v1/emails/<id>?format=full"
```

Response includes `ai` object with: `summary`, `labels`, `action`, `suggested_response`, `keywords`.

### Search Emails

```bash
curl -X POST -H "Authorization: Bearer ip_<key>" \
  -H "Content-Type: application/json" \
  -d '{"query": "invoice from Acme", "mode": "hybrid", "limit": 20}' \
  "https://inboxparse.com/api/v1/search"
```

Modes: `fulltext`, `semantic`, `hybrid` (default). Each search counts against quota.

### Send Email (Admin Key Required)

```bash
curl -X POST -H "Authorization: Bearer ip_<key>" \
  -H "Content-Type: application/json" \
  -d '{
    "mailbox_id": "<uuid>",
    "to": ["recipient@example.com"],
    "subject": "Hello",
    "body": "Plain text body"
  }' \
  "https://inboxparse.com/api/v1/emails/send"
```

Optional fields: `cc`, `bcc`, `html_body`, `reply_to`. Max 100 recipients, 10MB body.

### Reply to Email (Admin Key Required)

```bash
curl -X POST -H "Authorization: Bearer ip_<key>" \
  -H "Content-Type: application/json" \
  -d '{
    "email_id": "msg_<id>",
    "mailbox_id": "<uuid>",
    "body": "Reply text"
  }' \
  "https://inboxparse.com/api/v1/emails/reply"
```

Auto-populates To from original sender. Maintains thread references.

### Create Webhook

```bash
curl -X POST -H "Authorization: Bearer ip_<key>" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-app.com/webhook",
    "events": ["email.received", "email.ai_processed"],
    "auth_type": "hmac",
    "secret": "your_hmac_secret"
  }' \
  "https://inboxparse.com/api/v1/webhooks"
```

Events: `email.received`, `email.sent`, `email.ai_processed`, `mailbox.synced`, `mailbox.error`.

### Create Custom Label

```bash
curl -X POST -H "Authorization: Bearer ip_<key>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Urgent",
    "color": "#EF4444",
    "prompt": "Classify as Urgent if the email requires immediate action"
  }' \
  "https://inboxparse.com/api/v1/labels"
```

The `prompt` field (max 20000 chars) instructs the AI classifier.

---

## Pagination

Use cursor-based pagination for list endpoints:

```bash
# First page
curl -H "Authorization: Bearer ip_<key>" \
  "https://inboxparse.com/api/v1/emails?limit=20"

# Next page (use next_cursor from response)
curl -H "Authorization: Bearer ip_<key>" \
  "https://inboxparse.com/api/v1/emails?limit=20&cursor=<next_cursor>"
```

Continue until `pagination.has_more` is `false`.

---

## Error Handling

| Code | HTTP | Meaning |
|------|------|---------|
| `unauthorized` | 401 | Missing or invalid API key |
| `forbidden` | 403 | Member key used for write operation |
| `not_found` | 404 | Resource doesn't exist |
| `validation_error` | 400 | Invalid request body |
| `duplicate` | 409 | Unique constraint (e.g. label name) |
| `rate_limit_exceeded` | 429 | Quota exceeded for billing tier |
| `internal_error` | 500 | Server error |

Always check `error.code` in responses for programmatic handling.

---

## MCP Server

InboxParse also offers an MCP (Model Context Protocol) server for direct AI client integration:

- **URL**: `https://inboxparse.com/api/mcp`
- **Auth**: OAuth 2.1 with PKCE
- **Tools**: 15+ tools mapping to V1 endpoints
- **Scopes**: `email:read`, `thread:read`, `search`, `mailbox:read/write`, `label:read/write`, `webhook:read/write`, `usage:read`

Configure in Claude Desktop or other MCP clients for native email access.

---

## Tips for Agents

1. **Start with GET endpoints** — use member keys for read operations
2. **Use `format=markdown`** — best for LLM consumption (default)
3. **Use `format=full`** when you need HTML + text + markdown
4. **Search before listing** — `POST /search` with `mode=hybrid` is fastest for finding specific emails
5. **Check `ai` field** — emails include AI-generated summaries, labels, and suggested responses
6. **Monitor usage** — `GET /usage` returns current period stats and limits
7. **Webhook for real-time** — set up webhooks for `email.received` instead of polling
