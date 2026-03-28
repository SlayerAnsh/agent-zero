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

### TypeScript Example

```typescript
const BASE_URL = "http://localhost:50001";
const API_KEY = "your-api-key";

const headers = {
  "Content-Type": "application/json",
  "X-API-KEY": API_KEY,
};

interface MessageResponse {
  context_id: string;
  response: string;
}

interface LogResponse {
  context_id: string;
  log: {
    guid: string;
    total_items: number;
    returned_items: number;
    start_position: number;
    progress: string;
    progress_active: boolean;
    items: Array<{
      no: number;
      type: string;
      heading: string;
      content: string;
      kvps: Record<string, unknown>;
      id: string | null;
    }>;
  };
}

interface TerminateResponse {
  success: boolean;
  message: string;
  context_id: string;
}

async function sendMessage(
  message: string,
  contextId?: string,
  projectName?: string
): Promise<MessageResponse> {
  const body: Record<string, unknown> = { message };
  if (contextId) body.context_id = contextId;
  if (projectName) body.project_name = projectName;

  const resp = await fetch(`${BASE_URL}/api/api_message`, {
    method: "POST",
    headers,
    body: JSON.stringify(body),
  });

  if (!resp.ok) {
    const err = await resp.json();
    throw new Error(err.error ?? `HTTP ${resp.status}`);
  }
  return resp.json();
}

async function getLogs(
  contextId: string,
  length = 50
): Promise<LogResponse> {
  const resp = await fetch(`${BASE_URL}/api/api_log_get`, {
    method: "POST",
    headers,
    body: JSON.stringify({ context_id: contextId, length }),
  });
  return resp.json();
}

async function getFiles(
  paths: string[]
): Promise<Record<string, string>> {
  const resp = await fetch(`${BASE_URL}/api/api_files_get`, {
    method: "POST",
    headers,
    body: JSON.stringify({ paths }),
  });
  return resp.json();
}

async function terminateChat(
  contextId: string
): Promise<TerminateResponse> {
  const resp = await fetch(`${BASE_URL}/api/api_terminate_chat`, {
    method: "POST",
    headers,
    body: JSON.stringify({ context_id: contextId }),
  });
  return resp.json();
}

// ─── Usage ───────────────────────────────────────────────────────────
async function main() {
  // Start a conversation with a project
  const first = await sendMessage(
    "Analyze the codebase and list the main modules",
    undefined,
    "my-project"
  );
  const contextId = first.context_id;
  console.log("Agent:", first.response);

  // Continue the conversation
  const second = await sendMessage(
    "Now create a summary document",
    contextId
  );
  console.log("Agent:", second.response);

  // Get conversation logs
  const logs = await getLogs(contextId);
  for (const item of logs.log.items) {
    console.log(`[${item.type}] ${item.content.slice(0, 100)}`);
  }

  // Download generated files
  const files = await getFiles([
    "/a0/usr/projects/my-project/summary.md",
  ]);
  for (const [filename, b64] of Object.entries(files)) {
    const buf = Buffer.from(b64, "base64");
    const fs = await import("fs/promises");
    await fs.writeFile(filename, buf);
    console.log(`Downloaded: ${filename}`);
  }

  // Clean up
  await terminateChat(contextId);
}

main().catch(console.error);
```

### Go Example

```go
package main

import (
	"bytes"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
)

const (
	baseURL = "http://localhost:50001"
	apiKey  = "your-api-key"
)

// ─── Types ───────────────────────────────────────────────────────────

type MessageRequest struct {
	Message     string       `json:"message"`
	ContextID   string       `json:"context_id,omitempty"`
	ProjectName string       `json:"project_name,omitempty"`
	Attachments []Attachment `json:"attachments,omitempty"`
}

type Attachment struct {
	Filename string `json:"filename"`
	Base64   string `json:"base64"`
}

type MessageResponse struct {
	ContextID string `json:"context_id"`
	Response  string `json:"response"`
}

type LogResponse struct {
	ContextID string `json:"context_id"`
	Log       struct {
		GUID           string    `json:"guid"`
		TotalItems     int       `json:"total_items"`
		ReturnedItems  int       `json:"returned_items"`
		StartPosition  int       `json:"start_position"`
		Progress       string    `json:"progress"`
		ProgressActive bool      `json:"progress_active"`
		Items          []LogItem `json:"items"`
	} `json:"log"`
}

type LogItem struct {
	No      int                    `json:"no"`
	Type    string                 `json:"type"`
	Heading string                 `json:"heading"`
	Content string                 `json:"content"`
	KVPs    map[string]interface{} `json:"kvps"`
	ID      *string                `json:"id"`
}

type TerminateResponse struct {
	Success   bool   `json:"success"`
	Message   string `json:"message"`
	ContextID string `json:"context_id"`
}

// ─── HTTP helper ─────────────────────────────────────────────────────

func apiPost(path string, payload interface{}, result interface{}) error {
	body, err := json.Marshal(payload)
	if err != nil {
		return fmt.Errorf("marshal: %w", err)
	}

	req, err := http.NewRequest("POST", baseURL+path, bytes.NewReader(body))
	if err != nil {
		return fmt.Errorf("new request: %w", err)
	}
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("X-API-KEY", apiKey)

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return fmt.Errorf("do: %w", err)
	}
	defer resp.Body.Close()

	respBody, err := io.ReadAll(resp.Body)
	if err != nil {
		return fmt.Errorf("read body: %w", err)
	}

	if resp.StatusCode != 200 {
		return fmt.Errorf("HTTP %d: %s", resp.StatusCode, string(respBody))
	}

	return json.Unmarshal(respBody, result)
}

// ─── API functions ───────────────────────────────────────────────────

func sendMessage(message, contextID, projectName string) (*MessageResponse, error) {
	req := MessageRequest{
		Message:     message,
		ContextID:   contextID,
		ProjectName: projectName,
	}
	var res MessageResponse
	err := apiPost("/api/api_message", req, &res)
	return &res, err
}

func getLogs(contextID string, length int) (*LogResponse, error) {
	req := map[string]interface{}{
		"context_id": contextID,
		"length":     length,
	}
	var res LogResponse
	err := apiPost("/api/api_log_get", req, &res)
	return &res, err
}

func getFiles(paths []string) (map[string]string, error) {
	req := map[string]interface{}{"paths": paths}
	var res map[string]string
	err := apiPost("/api/api_files_get", req, &res)
	return res, err
}

func terminateChat(contextID string) (*TerminateResponse, error) {
	req := map[string]string{"context_id": contextID}
	var res TerminateResponse
	err := apiPost("/api/api_terminate_chat", req, &res)
	return &res, err
}

func resetChat(contextID string) error {
	req := map[string]string{"context_id": contextID}
	var res map[string]interface{}
	return apiPost("/api/api_reset_chat", req, &res)
}

// ─── Main ────────────────────────────────────────────────────────────

func main() {
	// Start a conversation with a project
	first, err := sendMessage(
		"Analyze the codebase and list the main modules",
		"",          // no context_id — creates new chat
		"my-project", // activate this project
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	contextID := first.ContextID
	fmt.Println("Agent:", first.Response)

	// Continue the conversation
	second, err := sendMessage("Now create a summary document", contextID, "")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	fmt.Println("Agent:", second.Response)

	// Get conversation logs
	logs, err := getLogs(contextID, 50)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	for _, item := range logs.Log.Items {
		content := item.Content
		if len(content) > 100 {
			content = content[:100]
		}
		fmt.Printf("[%s] %s\n", item.Type, content)
	}

	// Download generated files
	files, err := getFiles([]string{"/a0/usr/projects/my-project/summary.md"})
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	for filename, b64Content := range files {
		data, _ := base64.StdEncoding.DecodeString(b64Content)
		os.WriteFile(filename, data, 0644)
		fmt.Printf("Downloaded: %s\n", filename)
	}

	// Clean up
	terminateChat(contextID)
}
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

### Example 3: Two-Project Pipeline via HTTP API Only

No A2A setup required — uses only the HTTP API endpoints with API keys. Good for when both projects run on the same instance. Examples in Python, TypeScript, and Go.

#### Python

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

#### TypeScript

```typescript
const BASE_URL = "http://localhost:50001";
const API_KEY = "your-api-key";

const headers = {
  "Content-Type": "application/json",
  "X-API-KEY": API_KEY,
};

interface ChatResult {
  context_id: string;
  response: string;
}

async function chat(
  message: string,
  projectName?: string,
  contextId?: string
): Promise<ChatResult> {
  const body: Record<string, string> = { message };
  if (projectName) body.project_name = projectName;
  if (contextId) body.context_id = contextId;

  const resp = await fetch(`${BASE_URL}/api/api_message`, {
    method: "POST",
    headers,
    body: JSON.stringify(body),
  });
  if (!resp.ok) throw new Error(`HTTP ${resp.status}: ${await resp.text()}`);
  return resp.json();
}

async function terminate(contextId: string): Promise<void> {
  await fetch(`${BASE_URL}/api/api_terminate_chat`, {
    method: "POST",
    headers,
    body: JSON.stringify({ context_id: contextId }),
  });
}

async function main() {
  // Phase 1: Ask the "researcher" project to gather information
  const research = await chat(
    "Search the codebase for all database migration files and summarize the schema changes in the last 5 migrations.",
    "researcher"
  );
  const ctxResearch = research.context_id;
  console.log("[Researcher]", research.response.slice(0, 200), "...");

  // Phase 2: Feed research into the "writer" project
  const writer = await chat(
    `Based on the following schema change summary, write a changelog entry for the next release:\n\n${research.response}`,
    "writer"
  );
  const ctxWriter = writer.context_id;
  console.log("[Writer]", writer.response.slice(0, 200), "...");

  // Phase 3: Send the draft back to researcher for fact-checking
  const review = await chat(
    `Fact-check this changelog draft against the actual migration files. Flag any inaccuracies:\n\n${writer.response}`,
    undefined,
    ctxResearch // reuse the researcher's context
  );
  console.log("[Researcher Review]", review.response.slice(0, 200), "...");

  // Phase 4: Final edit by writer
  const final_ = await chat(
    `Apply these corrections to your draft and produce the final changelog:\n\n${review.response}`,
    undefined,
    ctxWriter // reuse the writer's context
  );
  console.log("[Final Changelog]");
  console.log(final_.response);

  // Clean up
  await terminate(ctxResearch);
  await terminate(ctxWriter);
  console.log("\nDone. Both contexts cleaned up.");
}

main().catch(console.error);
```

#### Go

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
)

const (
	baseURL = "http://localhost:50001"
	apiKey  = "your-api-key"
)

type chatResult struct {
	ContextID string `json:"context_id"`
	Response  string `json:"response"`
}

func apiPost(path string, payload interface{}, result interface{}) error {
	body, _ := json.Marshal(payload)
	req, _ := http.NewRequest("POST", baseURL+path, bytes.NewReader(body))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("X-API-KEY", apiKey)

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	respBody, _ := io.ReadAll(resp.Body)
	if resp.StatusCode != 200 {
		return fmt.Errorf("HTTP %d: %s", resp.StatusCode, respBody)
	}
	return json.Unmarshal(respBody, result)
}

func chat(message, projectName, contextID string) (*chatResult, error) {
	payload := map[string]string{"message": message}
	if projectName != "" {
		payload["project_name"] = projectName
	}
	if contextID != "" {
		payload["context_id"] = contextID
	}
	var res chatResult
	err := apiPost("/api/api_message", payload, &res)
	return &res, err
}

func terminate(contextID string) {
	var res map[string]interface{}
	apiPost("/api/api_terminate_chat", map[string]string{"context_id": contextID}, &res)
}

func truncate(s string, n int) string {
	if len(s) > n {
		return s[:n] + "..."
	}
	return s
}

func main() {
	// Phase 1: Ask the "researcher" project to gather information
	research, err := chat(
		"Search the codebase for all database migration files and summarize the schema changes in the last 5 migrations.",
		"researcher", "",
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	ctxResearch := research.ContextID
	fmt.Printf("[Researcher] %s\n", truncate(research.Response, 200))

	// Phase 2: Feed research into the "writer" project
	writer, err := chat(
		fmt.Sprintf("Based on the following schema change summary, write a changelog entry for the next release:\n\n%s", research.Response),
		"writer", "",
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	ctxWriter := writer.ContextID
	fmt.Printf("[Writer] %s\n", truncate(writer.Response, 200))

	// Phase 3: Send the draft back to researcher for fact-checking
	review, err := chat(
		fmt.Sprintf("Fact-check this changelog draft against the actual migration files. Flag any inaccuracies:\n\n%s", writer.Response),
		"", ctxResearch, // reuse the researcher's context
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("[Researcher Review] %s\n", truncate(review.Response, 200))

	// Phase 4: Final edit by writer
	final, err := chat(
		fmt.Sprintf("Apply these corrections to your draft and produce the final changelog:\n\n%s", review.Response),
		"", ctxWriter, // reuse the writer's context
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("[Final Changelog]\n%s\n", final.Response)

	// Clean up
	terminate(ctxResearch)
	terminate(ctxWriter)
	fmt.Println("\nDone. Both contexts cleaned up.")
}
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
