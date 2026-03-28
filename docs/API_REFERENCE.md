# Agent Zero API Reference

This document covers the external HTTP API endpoints that can be called using an API key to interact with the Agent Zero platform programmatically. These endpoints allow you to send messages to agents, manage chat sessions, work with projects, list agent profiles, and retrieve files and logs.

## Authentication

All external API endpoints require an API key passed via the `X-API-KEY` HTTP header.

```
X-API-KEY: <your-api-key>
```

Alternatively, the API key can be passed in the JSON request body as `api_key`:

```json
{
  "api_key": "<your-api-key>"
}
```

### How the API Key is Generated

The API key is automatically derived from three values:

- The persistent runtime ID of the Agent Zero instance
- The configured `AUTH_LOGIN` username
- The configured `AUTH_PASSWORD` password

A SHA-256 hash of `{runtime_id}:{username}:{password}` is computed, then base64url-encoded and truncated to 16 characters. This means:

- The token is deterministic — it stays the same as long as the credentials and runtime ID remain unchanged.
- It changes when you update the login credentials.
- You can find the current token in Settings under the field `mcp_server_token`.

### Base URL

All endpoints are relative to your Agent Zero instance URL:

```
http://<host>:<port>
```

---

## Chat / Message Endpoints

These are the primary endpoints for interacting with agents programmatically.

### Send Message

Send a message to an agent and receive the response synchronously. This is the main endpoint for programmatic interaction.

```
POST /api/api_message
```

**Authentication:** API Key required

**Request Body:**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `message` | string | Yes | — | The text message to send to the agent |
| `context_id` | string | No | "" | Chat context ID. Omit to create a new chat. Provide to continue an existing conversation |
| `attachments` | array | No | [] | File attachments as base64-encoded objects |
| `lifetime_hours` | number | No | 24 | Hours before the chat is automatically cleaned up |
| `project_name` | string | No | null | Project to activate for this chat (first message only) |
| `agent_profile` | string | No | null | Agent profile to use (first message only, cannot be changed on existing context) |

**Attachment format:**

```json
{
  "filename": "document.pdf",
  "base64": "<base64-encoded-file-content>"
}
```

**Success Response (200):**

```json
{
  "context_id": "aB3kLm9x",
  "response": "The agent's response text..."
}
```

**Error Responses:**

| Status | Body | Condition |
|--------|------|-----------|
| 400 | `{"error": "Message is required"}` | Empty message |
| 400 | `{"error": "Cannot override agent profile on existing context"}` | Trying to change agent profile on existing chat |
| 400 | `{"error": "Project can only be set on first message"}` | Trying to change project on existing chat |
| 404 | `{"error": "Context not found"}` | Invalid `context_id` |
| 500 | `{"error": "..."}` | Server error |

**Example — Start a new conversation:**

```bash
curl -X POST http://localhost:50001/api/api_message \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: your-api-key" \
  -d '{
    "message": "Hello, what can you help me with?",
    "project_name": "my-project"
  }'
```

**Example — Continue a conversation:**

```bash
curl -X POST http://localhost:50001/api/api_message \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: your-api-key" \
  -d '{
    "context_id": "aB3kLm9x",
    "message": "Now explain that in more detail"
  }'
```

**Example — Send with attachments:**

```bash
curl -X POST http://localhost:50001/api/api_message \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: your-api-key" \
  -d '{
    "message": "Analyze this file",
    "attachments": [
      {
        "filename": "data.csv",
        "base64": "Y29sMSxjb2wyLGNvbDMKMSwyLDMK"
      }
    ]
  }'
```

**Notes:**
- The call is synchronous — it blocks until the agent finishes processing and returns the full response.
- Expired chats (past `lifetime_hours`) are cleaned up automatically on subsequent API calls.
- The `context_id` returned should be saved and reused to maintain conversation continuity.

---

### Reset Chat

Reset an existing chat context. Clears all conversation history but keeps the context alive so you can send new messages to the same `context_id`.

```
POST /api/api_reset_chat
```

**Authentication:** API Key required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `context_id` | string | Yes | The chat context to reset |

**Success Response (200):**

```json
{
  "success": true,
  "message": "Chat reset successfully",
  "context_id": "aB3kLm9x"
}
```

**Error Responses:**

| Status | Body | Condition |
|--------|------|-----------|
| 400 | `{"error": "context_id is required"}` | Missing context_id |
| 404 | `{"error": "Chat context not found"}` | Invalid context_id |

---

### Terminate Chat

Permanently delete a chat context and all its data.

```
POST /api/api_terminate_chat
```

**Authentication:** API Key required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `context_id` | string | Yes | The chat context to delete |

**Success Response (200):**

```json
{
  "success": true,
  "message": "Chat deleted successfully",
  "context_id": "aB3kLm9x"
}
```

**Error Responses:**

| Status | Body | Condition |
|--------|------|-----------|
| 400 | `{"error": "context_id is required"}` | Missing context_id |
| 404 | `{"error": "Chat context not found"}` | Invalid context_id |

---

### Get Chat Logs

Retrieve log entries from a chat context. Useful for reviewing the conversation history and agent activity.

```
GET /api/api_log_get?context_id=<id>&length=<n>
POST /api/api_log_get
```

**Authentication:** API Key required

**Parameters (query string for GET, JSON body for POST):**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `context_id` | string | Yes | — | The chat context to retrieve logs from |
| `length` | number | No | 100 | Number of most recent log entries to return |

**Success Response (200):**

```json
{
  "context_id": "aB3kLm9x",
  "log": {
    "guid": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "total_items": 42,
    "returned_items": 42,
    "start_position": 0,
    "progress": "Processing...",
    "progress_active": true,
    "items": [
      {
        "no": 0,
        "type": "user",
        "heading": "",
        "content": "Hello",
        "kvps": {},
        "id": "msg-uuid"
      },
      {
        "no": 1,
        "type": "response",
        "heading": "Agent Response",
        "content": "Hello! How can I help you?",
        "kvps": {},
        "id": null
      }
    ]
  }
}
```

**Log item types:** `user`, `agent`, `response`, `tool`, `code_exe`, `browser`, `subagent`, `error`, `warning`, `info`, `hint`, `progress`, `mcp`, `input`, `util`

---

### Get Files

Retrieve files from the Agent Zero filesystem as base64-encoded content. Useful for downloading files generated by the agent.

```
POST /api/api_files_get
```

**Authentication:** API Key required

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `paths` | array of strings | Yes | File paths to retrieve. Supports internal paths (starting with `/a0/`) and absolute paths |

**Path formats:**
- Internal upload path: `/a0/tmp/uploads/file.txt` or `/a0/usr/uploads/file.txt`
- Internal Agent Zero path: `/a0/some/path/file.txt`
- Absolute filesystem path: `/absolute/path/to/file.txt`

**Success Response (200):**

A dictionary mapping filenames to their base64-encoded content:

```json
{
  "report.pdf": "JVBERi0xLjQK...",
  "data.csv": "Y29sMSxjb2wyLGNvbDMK..."
}
```

Files that don't exist are silently skipped.

---

## Project Endpoints

Projects provide isolated workspaces with their own instructions, knowledge files, environment variables, and git integration. The project API uses a single endpoint with an `action` field to determine the operation.

```
POST /api/projects
```

**Authentication:** Session-based (web UI auth + CSRF token). Not available via API key by default.

**Common Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | The operation to perform |
| `context_id` | string | No | Chat context ID (required for activate/deactivate) |

### List Projects

Get all available projects.

**Request:**

```json
{
  "action": "list"
}
```

**Response:**

```json
{
  "ok": true,
  "data": [
    {
      "name": "my-project",
      "title": "My Project",
      "description": "Project description",
      "color": "blue"
    }
  ]
}
```

### List Project Options

Get projects as key/label pairs (useful for dropdown menus).

**Request:**

```json
{
  "action": "list_options"
}
```

**Response:**

```json
{
  "ok": true,
  "data": [
    {"key": "my-project", "label": "My Project"}
  ]
}
```

### Load Project Details

Get full project details for editing.

**Request:**

```json
{
  "action": "load",
  "name": "my-project"
}
```

**Response:**

```json
{
  "ok": true,
  "data": {
    "name": "my-project",
    "title": "My Project",
    "description": "A detailed description",
    "instructions": "System instructions for the agent...",
    "color": "blue",
    "git_url": "https://github.com/user/repo.git",
    "file_structure": {
      "enabled": true,
      "max_depth": 5,
      "max_files": 20,
      "max_folders": 20,
      "max_lines": 250,
      "gitignore": ""
    },
    "instruction_files_count": 3,
    "knowledge_files_count": 5,
    "variables": "KEY=value\nKEY2=value2",
    "secrets": "API_KEY=****",
    "subagents": {},
    "git_status": {
      "is_git_repo": true,
      "remote_url": "https://github.com/user/repo.git",
      "current_branch": "main",
      "is_dirty": false,
      "untracked_count": 0,
      "last_commit": {}
    }
  }
}
```

### Create Project

**Request:**

```json
{
  "action": "create",
  "project": {
    "name": "new-project",
    "title": "New Project",
    "description": "What this project is about",
    "instructions": "You are an assistant for...",
    "color": "green",
    "git_url": "",
    "file_structure": {
      "enabled": true,
      "max_depth": 5,
      "max_files": 20,
      "max_folders": 20,
      "max_lines": 250,
      "gitignore": ""
    }
  }
}
```

**Response:** Returns the full project data (same structure as Load).

### Clone Git Project

Create a project by cloning a git repository.

**Request:**

```json
{
  "action": "clone",
  "project": {
    "name": "cloned-project",
    "title": "Cloned Project",
    "description": "",
    "instructions": "",
    "color": "purple",
    "git_url": "https://github.com/user/repo.git",
    "git_token": "ghp_xxxxxxxxxxxx",
    "file_structure": {
      "enabled": true,
      "max_depth": 5,
      "max_files": 20,
      "max_folders": 20,
      "max_lines": 250,
      "gitignore": ""
    }
  }
}
```

### Update Project

**Request:**

```json
{
  "action": "update",
  "project": {
    "name": "my-project",
    "title": "Updated Title",
    "description": "Updated description",
    "instructions": "Updated instructions...",
    "color": "red",
    "git_url": "",
    "file_structure": {
      "enabled": true,
      "max_depth": 5,
      "max_files": 20,
      "max_folders": 20,
      "max_lines": 250,
      "gitignore": ""
    },
    "instruction_files_count": 3,
    "knowledge_files_count": 5,
    "variables": "KEY=value",
    "secrets": "SECRET=value",
    "subagents": {}
  }
}
```

### Delete Project

**Request:**

```json
{
  "action": "delete",
  "name": "my-project"
}
```

### Activate Project for a Chat

Associate a project with a chat context. The agent will use the project's instructions, knowledge, and environment.

**Request:**

```json
{
  "action": "activate",
  "context_id": "aB3kLm9x",
  "name": "my-project"
}
```

### Deactivate Project

Remove the project association from a chat context.

**Request:**

```json
{
  "action": "deactivate",
  "context_id": "aB3kLm9x"
}
```

### Get File Structure

Preview the file tree of a project directory.

**Request:**

```json
{
  "action": "file_structure",
  "name": "my-project",
  "settings": {
    "enabled": true,
    "max_depth": 3,
    "max_files": 10,
    "max_folders": 10,
    "max_lines": 100,
    "gitignore": "node_modules\n.git"
  }
}
```

**All project responses follow this envelope:**

```json
{
  "ok": true,
  "data": { ... }
}
```

```json
{
  "ok": false,
  "error": "Error message"
}
```

---

## Agent Endpoints

### List Agent Profiles

Returns all available agent profiles from default, user, plugin, and project sources.

```
POST /api/agents
```

**Authentication:** Session-based (web UI auth + CSRF token)

**Request:**

```json
{
  "action": "list"
}
```

**Response:**

```json
{
  "ok": true,
  "data": [
    {
      "name": "default",
      "title": "Default Agent",
      "description": "General purpose assistant",
      "context": "",
      "path": "/path/to/agents/default",
      "origin": ["default"],
      "enabled": true
    },
    {
      "name": "coder",
      "title": "Coding Agent",
      "description": "Specialized for programming tasks",
      "context": "",
      "path": "/path/to/agents/coder",
      "origin": ["user"],
      "enabled": true
    }
  ]
}
```

**Agent profile fields:**

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Unique agent profile identifier |
| `title` | string | Display name (defaults to `name` if empty) |
| `description` | string | What this agent profile specializes in |
| `context` | string | Context information |
| `path` | string | Filesystem path to the agent profile directory |
| `origin` | array | Source(s): `"default"`, `"user"`, `"plugin"`, `"project"` |
| `enabled` | bool | Whether the agent profile is active |

**Agent profile origins (in merge priority order):**
1. `default` — Built-in agents in `agents/` directory
2. `plugin` — Agents from installed plugins
3. `user` — Custom agents in `usr/agents/` directory
4. `project` — Project-specific agents defined in `.a0proj/agents.json`

---

## Typical Workflow

Here is a typical sequence of API calls for a programmatic integration:

```
1. POST /api/api_message  (start conversation, get context_id)
        ↓
2. POST /api/api_message  (continue with context_id)
        ↓
3. POST /api/api_log_get  (review full activity log)
        ↓
4. POST /api/api_files_get  (download generated files)
        ↓
5. POST /api/api_reset_chat  (clear history, reuse context)
   — or —
   POST /api/api_terminate_chat  (delete context entirely)
```

### Python Example

```python
import requests

BASE_URL = "http://localhost:50001"
API_KEY = "your-api-key"
HEADERS = {
    "Content-Type": "application/json",
    "X-API-KEY": API_KEY,
}

# Start a new conversation with a project
resp = requests.post(f"{BASE_URL}/api/api_message", headers=HEADERS, json={
    "message": "Analyze the codebase and list the main modules",
    "project_name": "my-project",
})
data = resp.json()
context_id = data["context_id"]
print("Agent:", data["response"])

# Continue the conversation
resp = requests.post(f"{BASE_URL}/api/api_message", headers=HEADERS, json={
    "context_id": context_id,
    "message": "Now create a summary document",
})
data = resp.json()
print("Agent:", data["response"])

# Get conversation logs
resp = requests.post(f"{BASE_URL}/api/api_log_get", headers=HEADERS, json={
    "context_id": context_id,
    "length": 50,
})
logs = resp.json()
for item in logs["log"]["items"]:
    print(f"[{item['type']}] {item['content'][:100]}")

# Download generated files
resp = requests.post(f"{BASE_URL}/api/api_files_get", headers=HEADERS, json={
    "paths": ["/a0/usr/projects/my-project/summary.md"],
})
import base64
for filename, b64content in resp.json().items():
    with open(filename, "wb") as f:
        f.write(base64.b64decode(b64content))

# Clean up
requests.post(f"{BASE_URL}/api/api_terminate_chat", headers=HEADERS, json={
    "context_id": context_id,
})
```

### cURL Quick Reference

```bash
# Start conversation
curl -X POST http://localhost:50001/api/api_message \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: your-key" \
  -d '{"message": "Hello"}'

# Continue conversation
curl -X POST http://localhost:50001/api/api_message \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: your-key" \
  -d '{"context_id": "aB3kLm9x", "message": "Tell me more"}'

# Get logs
curl -X POST http://localhost:50001/api/api_log_get \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: your-key" \
  -d '{"context_id": "aB3kLm9x", "length": 20}'

# Get files
curl -X POST http://localhost:50001/api/api_files_get \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: your-key" \
  -d '{"paths": ["/a0/usr/uploads/output.txt"]}'

# Reset chat
curl -X POST http://localhost:50001/api/api_reset_chat \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: your-key" \
  -d '{"context_id": "aB3kLm9x"}'

# Delete chat
curl -X POST http://localhost:50001/api/api_terminate_chat \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: your-key" \
  -d '{"context_id": "aB3kLm9x"}'
```

---

## Agent-to-Agent (A2A) Protocol

Agent Zero also exposes an A2A endpoint for inter-agent communication. This uses the same API key (token) embedded in the URL path.

```
POST /a2a/t-<API_TOKEN>
POST /a2a/t-<API_TOKEN>/p-<PROJECT_NAME>
```

The A2A protocol follows the FastA2A message format with `role`, `parts` (text, file), `message_id`, and optional `context_id`. This is primarily used for connecting multiple Agent Zero instances together.

**Connection URL format:**

```
http://<host>:<port>/a2a/t-<API_TOKEN>
http://<host>:<port>/a2a/t-<API_TOKEN>/p-<PROJECT_NAME>
```

---

## Notes

- **Rate limits:** There are no built-in API rate limits, but the underlying LLM providers have their own rate limits configured per model.
- **Concurrency:** Each `context_id` processes one message at a time. Sending a message while the agent is busy will queue it.
- **Chat expiry:** Chats created via the external API have a configurable lifetime (`lifetime_hours`, default 24). Expired chats are cleaned up automatically.
- **File paths:** Files generated by the agent are stored under `/a0/usr/` internally. Use the `/api/api_files_get` endpoint to retrieve them.
- **Project storage:** Projects are stored on disk at `usr/projects/{project_name}/` with metadata in `.a0proj/`.
