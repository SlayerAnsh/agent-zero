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

Agent Zero implements the [FastA2A](https://github.com/pydantic/fasta2a) protocol, allowing multiple Agent Zero instances (or any A2A-compatible agent) to communicate with each other. The A2A server must be enabled in Settings (`a2a_server_enabled: true`).

### Prerequisites

- The `fasta2a` Python package must be installed (the server returns 503 if unavailable).
- A2A server must be enabled in Settings.
- The API token (same `mcp_server_token` used for the HTTP API) is used for authentication.

### A2A Endpoint URLs

Authentication is done via the token embedded in the URL path:

```
POST /a2a/t-<API_TOKEN>
POST /a2a/t-<API_TOKEN>/p-<PROJECT_NAME>
```

Alternative authentication (when token is not in URL path):
- Header: `Authorization: Bearer <API_TOKEN>`
- Header: `X-API-KEY: <API_TOKEN>`
- Query param: `?api_key=<API_TOKEN>`

### Agent Card Discovery

Every A2A-compliant agent exposes a card at:

```
GET /.well-known/agent.json
```

This returns the agent's capabilities, skills, and metadata:

```json
{
  "name": "Agent Zero",
  "description": "A general AI assistant that can execute code, manage files, browse the web, and solve complex problems in an isolated Linux environment.",
  "version": "1.0.0",
  "provider": {
    "organization": "Agent Zero",
    "url": "https://github.com/frdel/agent-zero"
  },
  "skills": [
    {
      "id": "general_assistance",
      "name": "General AI Assistant",
      "description": "Provides general AI assistance including code execution, file management, web browsing, and problem solving",
      "tags": ["ai", "assistant", "code", "files", "web", "automation"],
      "examples": [
        "Write and execute Python code",
        "Manage files and directories",
        "Browse the web and extract information"
      ],
      "input_modes": ["text/plain", "application/octet-stream"],
      "output_modes": ["text/plain", "application/json"]
    }
  ]
}
```

### A2A Message Format

Messages follow the A2A specification with `role`, `parts`, and optional `metadata`:

**Sending a message (client → server):**

```json
{
  "role": "user",
  "parts": [
    {"kind": "text", "text": "Analyze this data"},
    {"kind": "file", "file": {"uri": "/path/to/file.csv"}}
  ],
  "kind": "message",
  "message_id": "unique-uuid",
  "context_id": "optional-existing-context"
}
```

**Receiving a response (server → client):**

```json
{
  "role": "agent",
  "parts": [
    {"kind": "text", "text": "Here is the analysis..."}
  ],
  "kind": "message",
  "message_id": "response-uuid"
}
```

### A2A Task Lifecycle

Each message sent creates a **task** that goes through these states:

| State | Description |
|-------|-------------|
| `submitted` | Task received and queued |
| `working` | Agent is processing the message |
| `completed` | Agent finished, response available |
| `failed` | Processing error occurred |
| `canceled` | Task was cancelled |

### Project Injection via URL

When you include `/p-<PROJECT_NAME>` in the URL path, the project name is automatically injected into the message's `metadata.project` field. The A2A worker then activates that project for the conversation context.

### A2A Behavior Notes

- Each A2A message creates a **temporary background context** (`AgentContextType.BACKGROUND`).
- The context is automatically **cleaned up after completion** — no manual termination needed.
- There is no persistent conversation across A2A calls (each call is stateless on the server side).
- The A2A server uses in-memory storage and broker — task state is lost on restart.

### Python A2A Client

Agent Zero includes a built-in A2A client (`helpers/fasta2a_client.py`) for connecting to remote agents:

```python
from helpers.fasta2a_client import AgentConnection

async with AgentConnection("http://remote-host:50001/a2a/t-THEIR_TOKEN") as conn:
    # Send a message
    response = await conn.send_message(
        message="Summarize the project status",
        attachments=["/path/to/report.pdf"],       # optional file URIs
        metadata={"project": "target-project"}     # optional project
    )

    # Extract result
    task = response["result"]
    task_id = task["id"]

    # If not immediately completed, poll for result
    final = await conn.wait_for_completion(task_id, poll_interval=2, max_wait=300)

    # Get the agent's response text
    history = final["result"]["history"]
    reply = history[-1]["parts"][0]["text"]
    print(reply)
```

**`AgentConnection` methods:**

| Method | Description |
|--------|-------------|
| `get_agent_card()` | Fetch the remote agent's capabilities card |
| `send_message(message, attachments?, context_id?, metadata?)` | Send a message, returns task info |
| `get_task(task_id)` | Check task status |
| `wait_for_completion(task_id, poll_interval=2, max_wait=300)` | Poll until task finishes |
| `close()` | Close the HTTP connection |

**Authentication options for the client:**

```python
# Token in URL path (recommended)
conn = AgentConnection("http://host:50001/a2a/t-TOKEN_HERE")

# Token as constructor argument (sent via Authorization + X-API-KEY headers)
conn = AgentConnection("http://host:50001/a2a", token="TOKEN_HERE")

# Token from environment variable A2A_TOKEN (fallback)
conn = AgentConnection("http://host:50001/a2a")
```

---

## End-to-End Examples

### Example 1: Single Project Workflow

Create a project, chat with the agent in that project's context, get the response, then deactivate and clean up.

> **Note:** Project creation requires session auth (web UI). The example below uses the API key endpoints for chat and the session endpoints for project management. In practice, you would create projects via the web UI and then use the API key for chat.

```python
import requests

BASE_URL = "http://localhost:50001"
API_KEY = "your-api-key"

HEADERS = {
    "Content-Type": "application/json",
    "X-API-KEY": API_KEY,
}

# For session-authenticated endpoints (projects), you need a session.
# This helper logs in and maintains cookies + CSRF token.
session = requests.Session()

def login_and_get_csrf():
    """Authenticate and retrieve a CSRF token for session-based endpoints."""
    # Log in
    session.post(f"{BASE_URL}/login", data={
        "username": "your-username",
        "password": "your-password",
    })
    # Get CSRF token
    resp = session.get(f"{BASE_URL}/api/csrf_token")
    csrf_data = resp.json()
    return csrf_data["token"]

csrf_token = login_and_get_csrf()
session_headers = {
    "Content-Type": "application/json",
    "X-CSRF-Token": csrf_token,
}


# ─── Step 1: Create a project ────────────────────────────────────────
resp = session.post(f"{BASE_URL}/api/projects", headers=session_headers, json={
    "action": "create",
    "project": {
        "name": "data-analysis",
        "title": "Data Analysis Project",
        "description": "Automated data analysis pipeline",
        "instructions": "You are a data analyst. Analyze CSV files and produce summary statistics.",
        "color": "blue",
        "git_url": "",
        "file_structure": {
            "enabled": True,
            "max_depth": 5,
            "max_files": 20,
            "max_folders": 20,
            "max_lines": 250,
            "gitignore": "",
        },
    },
})
project = resp.json()
print(f"Created project: {project['data']['name']}")


# ─── Step 2: Start a chat with the project ───────────────────────────
resp = requests.post(f"{BASE_URL}/api/api_message", headers=HEADERS, json={
    "message": "Load the sales data from /a0/usr/projects/data-analysis/sales.csv and show top 5 products by revenue.",
    "project_name": "data-analysis",
})
data = resp.json()
context_id = data["context_id"]
print(f"Context: {context_id}")
print(f"Agent: {data['response']}")


# ─── Step 3: Follow-up message in the same chat ──────────────────────
resp = requests.post(f"{BASE_URL}/api/api_message", headers=HEADERS, json={
    "context_id": context_id,
    "message": "Now create a bar chart of those results and save it as chart.png",
})
data = resp.json()
print(f"Agent: {data['response']}")


# ─── Step 4: Retrieve the generated file ─────────────────────────────
import base64

resp = requests.post(f"{BASE_URL}/api/api_files_get", headers=HEADERS, json={
    "paths": ["/a0/usr/projects/data-analysis/chart.png"],
})
for filename, b64content in resp.json().items():
    with open(filename, "wb") as f:
        f.write(base64.b64decode(b64content))
    print(f"Downloaded: {filename}")


# ─── Step 5: Review the full activity log ─────────────────────────────
resp = requests.post(f"{BASE_URL}/api/api_log_get", headers=HEADERS, json={
    "context_id": context_id,
    "length": 50,
})
logs = resp.json()
print(f"\n--- Activity Log ({logs['log']['total_items']} items) ---")
for item in logs["log"]["items"]:
    print(f"  [{item['type']:10s}] {item.get('content', '')[:80]}")


# ─── Step 6: Deactivate the project from the chat ────────────────────
resp = session.post(f"{BASE_URL}/api/projects", headers=session_headers, json={
    "action": "deactivate",
    "context_id": context_id,
})
print(f"\nProject deactivated: {resp.json()['ok']}")


# ─── Step 7: Clean up the chat ───────────────────────────────────────
resp = requests.post(f"{BASE_URL}/api/api_terminate_chat", headers=HEADERS, json={
    "context_id": context_id,
})
print(f"Chat terminated: {resp.json()['success']}")


# ─── Optional: Delete the project entirely ────────────────────────────
resp = session.post(f"{BASE_URL}/api/projects", headers=session_headers, json={
    "action": "delete",
    "name": "data-analysis",
})
print(f"Project deleted: {resp.json()['ok']}")
```

### Example 2: Two Projects with Agent-to-Agent Communication

This example shows two Agent Zero instances (or one instance with two projects) where:
- **Project A** ("frontend-app") handles frontend questions
- **Project B** ("backend-api") handles backend questions
- An external orchestrator sends a task to Project A, which then delegates a subtask to Project B via the A2A protocol

**Architecture:**

```
                          ┌──────────────────────┐
                          │   Your Application   │
                          │   (orchestrator)      │
                          └──────┬───────────────┘
                                 │  HTTP API
                    ┌────────────┴────────────┐
                    ▼                          ▼
          ┌─────────────────┐       ┌─────────────────┐
          │  Instance A      │       │  Instance B      │
          │  :50001          │◄─────►│  :50002          │
          │  frontend-app    │  A2A  │  backend-api     │
          └─────────────────┘       └─────────────────┘
```

**Step 1: Set up both instances**

Instance A runs on port 50001, Instance B on port 50002. Both have A2A enabled in Settings (`a2a_server_enabled: true`).

**Step 2: Orchestrator script**

```python
import requests
import asyncio
from helpers.fasta2a_client import AgentConnection

# ─── Configuration ────────────────────────────────────────────────────
INSTANCE_A_URL = "http://localhost:50001"
INSTANCE_B_URL = "http://localhost:50002"
TOKEN_A = "instance-a-api-key"
TOKEN_B = "instance-b-api-key"

HEADERS_A = {"Content-Type": "application/json", "X-API-KEY": TOKEN_A}
HEADERS_B = {"Content-Type": "application/json", "X-API-KEY": TOKEN_B}


# ─── Step 1: Send task to Frontend Agent (Instance A) ────────────────
print("=== Talking to Frontend Agent ===")
resp = requests.post(f"{INSTANCE_A_URL}/api/api_message", headers=HEADERS_A, json={
    "message": "Review the login page component and list all API endpoints it calls.",
    "project_name": "frontend-app",
})
frontend_data = resp.json()
context_a = frontend_data["context_id"]
print(f"Frontend Agent: {frontend_data['response']}")


# ─── Step 2: Send task to Backend Agent (Instance B) ─────────────────
# Use the frontend agent's findings to ask the backend agent about those endpoints
print("\n=== Talking to Backend Agent ===")
resp = requests.post(f"{INSTANCE_B_URL}/api/api_message", headers=HEADERS_B, json={
    "message": f"The frontend login page calls these endpoints. Check if they are all implemented and document any missing ones:\n\n{frontend_data['response']}",
    "project_name": "backend-api",
})
backend_data = resp.json()
context_b = backend_data["context_id"]
print(f"Backend Agent: {backend_data['response']}")


# ─── Step 3: Feed backend results back to frontend agent ─────────────
print("\n=== Sending backend findings back to Frontend Agent ===")
resp = requests.post(f"{INSTANCE_A_URL}/api/api_message", headers=HEADERS_A, json={
    "context_id": context_a,
    "message": f"The backend team reviewed your endpoint list. Here is their report. Update the login component to handle any missing endpoints gracefully:\n\n{backend_data['response']}",
})
final_data = resp.json()
print(f"Frontend Agent: {final_data['response']}")


# ─── Step 4: Clean up both chats ─────────────────────────────────────
requests.post(f"{INSTANCE_A_URL}/api/api_terminate_chat", headers=HEADERS_A, json={"context_id": context_a})
requests.post(f"{INSTANCE_B_URL}/api/api_terminate_chat", headers=HEADERS_B, json={"context_id": context_b})
print("\nBoth chats cleaned up.")
```

**Step 3: Direct A2A communication (agent-to-agent without orchestrator)**

If you want Instance A's agent to directly talk to Instance B during its own processing (without an external orchestrator), you can use the built-in `a2a_chat` tool. Configure Instance A's project instructions to include the connection info:

```
When you need backend API information, use the a2a_chat tool to ask the backend agent:
- Connection URL: http://localhost:50002/a2a/t-INSTANCE_B_TOKEN/p-backend-api
```

Or use the A2A client directly in a Python script:

```python
import asyncio
from helpers.fasta2a_client import AgentConnection


async def a2a_cross_project_chat():
    """
    Direct agent-to-agent communication between two Agent Zero instances.
    Instance A asks Instance B a question via the A2A protocol.
    """

    # ─── Connect to Instance B's A2A endpoint ─────────────────────────
    # Token is embedded in the URL; /p-backend-api activates the project
    a2a_url = "http://localhost:50002/a2a/t-INSTANCE_B_TOKEN/p-backend-api"

    async with AgentConnection(a2a_url) as remote_agent:
        # Check what the remote agent can do
        card = await remote_agent.get_agent_card()
        print(f"Connected to: {card['name']}")
        print(f"Skills: {[s['name'] for s in card.get('skills', [])]}")

        # ─── Send first message ───────────────────────────────────────
        print("\n--- Message 1 ---")
        response = await remote_agent.send_message(
            message="List all REST API endpoints in the authentication module"
        )
        task = response["result"]
        print(f"Task ID: {task['id']}")
        print(f"State: {task['status']['state']}")

        # If the response is already completed (blocking mode)
        if task["status"]["state"] == "completed":
            reply = task["history"][-1]["parts"][0]["text"]
            print(f"Agent reply: {reply[:200]}...")
        else:
            # Poll until done
            final = await remote_agent.wait_for_completion(
                task["id"], poll_interval=2, max_wait=120
            )
            reply = final["result"]["history"][-1]["parts"][0]["text"]
            print(f"Agent reply: {reply[:200]}...")

        # ─── Send follow-up (note: A2A is stateless per-call on server) ─
        # Each A2A call creates a fresh context, so include full context
        print("\n--- Message 2 ---")
        response2 = await remote_agent.send_message(
            message=f"Given these endpoints:\n{reply}\n\nWhich ones require JWT tokens and which use API keys?"
        )
        task2 = response2["result"]

        if task2["status"]["state"] == "completed":
            reply2 = task2["history"][-1]["parts"][0]["text"]
            print(f"Agent reply: {reply2[:200]}...")


# Run it
asyncio.run(a2a_cross_project_chat())
```

### Example 3: Full Async Two-Project Pipeline via HTTP API Only

No A2A setup required — uses only the HTTP API endpoints with API keys. Good for when both projects run on the same instance.

```python
import requests

BASE_URL = "http://localhost:50001"
API_KEY = "your-api-key"
HEADERS = {"Content-Type": "application/json", "X-API-KEY": API_KEY}


def chat(message, project_name=None, context_id=None):
    """Send a message and return (context_id, response)."""
    payload = {"message": message}
    if project_name:
        payload["project_name"] = project_name
    if context_id:
        payload["context_id"] = context_id
    resp = requests.post(f"{BASE_URL}/api/api_message", headers=HEADERS, json=payload)
    data = resp.json()
    return data["context_id"], data["response"]


def terminate(context_id):
    """Delete a chat context."""
    requests.post(f"{BASE_URL}/api/api_terminate_chat", headers=HEADERS, json={
        "context_id": context_id,
    })


# ─── Phase 1: Ask the "researcher" project to gather information ──────
ctx_research, research_output = chat(
    message="Search the codebase for all database migration files and summarize the schema changes in the last 5 migrations.",
    project_name="researcher",
)
print(f"[Researcher] {research_output[:200]}...")

# ─── Phase 2: Feed research into the "writer" project ─────────────────
ctx_writer, writer_output = chat(
    message=f"Based on the following schema change summary, write a changelog entry for the next release:\n\n{research_output}",
    project_name="writer",
)
print(f"[Writer] {writer_output[:200]}...")

# ─── Phase 3: Send the draft back to researcher for fact-checking ─────
_, review_output = chat(
    message=f"Fact-check this changelog draft against the actual migration files. Flag any inaccuracies:\n\n{writer_output}",
    context_id=ctx_research,  # reuse the researcher's context (it has the files loaded)
)
print(f"[Researcher Review] {review_output[:200]}...")

# ─── Phase 4: Final edit by writer ────────────────────────────────────
_, final_output = chat(
    message=f"Apply these corrections to your draft and produce the final changelog:\n\n{review_output}",
    context_id=ctx_writer,  # reuse the writer's context
)
print(f"[Final Changelog]\n{final_output}")

# ─── Clean up ─────────────────────────────────────────────────────────
terminate(ctx_research)
terminate(ctx_writer)
print("\nDone. Both contexts cleaned up.")
```

---

## Notes

- **Rate limits:** There are no built-in API rate limits, but the underlying LLM providers have their own rate limits configured per model.
- **Concurrency:** Each `context_id` processes one message at a time. Sending a message while the agent is busy will queue it.
- **Chat expiry:** Chats created via the external API have a configurable lifetime (`lifetime_hours`, default 24). Expired chats are cleaned up automatically.
- **File paths:** Files generated by the agent are stored under `/a0/usr/` internally. Use the `/api/api_files_get` endpoint to retrieve them.
- **Project storage:** Projects are stored on disk at `usr/projects/{project_name}/` with metadata in `.a0proj/`.
- **A2A is stateless:** Each A2A call creates a temporary background context that is destroyed after the response. There is no persistent conversation across A2A calls.
- **A2A requires `fasta2a` package:** The A2A server returns 503 if the package is not installed. The client (`AgentConnection`) raises `RuntimeError` if unavailable.
