---
name: filebrowser-manager
description: Manage files and folders on a Filebrowser server via HTTP API. Use when asked to list, move, delete, create, or edit files/folders on a Filebrowser instance. Works for any AI model.
---

# Filebrowser Manager

This skill provides a standardized way for any AI model to manage files and folders on a Filebrowser server by making direct HTTP calls. Filebrowser uses a "hidden" API with JWT authentication.

- **Official Repository**: [filebrowser/filebrowser](https://github.com/filebrowser/filebrowser)

## Inputs & Configuration

- **Host**: The Filebrowser URL (e.g., `https://filebrowser.example.com`)
- **Username**: Default is `""` (empty string)
- **Password**: Default is `""` (empty string)

## Authentication

All operations (except login) require the `X-Auth: [JWT_TOKEN]` header. Tokens expire in 2 hours.

### Login (Obtain JWT)

```bash
curl -X POST "https://[HOST]/api/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"[USERNAME]","password":"[PASSWORD]","recaptcha":""}'
```

**Output**: Returns the raw JWT token string directly in the response body.

---

## Core Operations

### 1. List Directory Contents

**Endpoint**: `GET /api/resources/[PATH]`
**Example**: `curl -X GET "https://[HOST]/api/resources/disk1" -H "X-Auth: [JWT_TOKEN]"`
**Presentation**: Parse the JSON `items` array and display as a clean table (Name, Size, Type, Modified).

### 2. Move or Rename

**Endpoint**: `PATCH /api/resources/[SOURCE_ENCODED]?action=rename&destination=[DEST_ENCODED]&override=false&rename=false`

- **Source ([SOURCE_ENCODED])**: Path relative to root. **Special characters MUST be percent-encoded, but slashes (`/`) MUST NOT be encoded.**
- **Destination ([DEST_ENCODED])**: **MUST** be fully percent-encoded (including slashes).

**Example**: Move `disk2/downloads/test 1/` to `disk2/test 1/`

```bash
curl -X PATCH "https://[HOST]/api/resources/disk2/downloads/test%201/?action=rename&destination=%2Fdisk2%2Ftest%25201&override=false&rename=false" -H "X-Auth: [JWT_TOKEN]"
```

### 3. Create Folder

**Endpoint**: `POST /api/resources/[PATH]/[NAME]/?override=false`
**Rule**: The path **MUST** end with a trailing slash (`/`).
**Example**: `curl -X POST "https://[HOST]/api/resources/disk2/newfolder/?override=false" -H "X-Auth: [JWT_TOKEN]"`

### 4. Create Empty File

**Endpoint**: `POST /api/resources/[PATH]/[NAME]?override=false`
**Rule**: The path **MUST NOT** end with a trailing slash.
**Example**: `curl -X POST "https://[HOST]/api/resources/disk2/newfile.txt?override=false" -H "X-Auth: [JWT_TOKEN]"`

### 5. Delete File/Folder

**Endpoint**: `DELETE /api/resources/[PATH]`
**Example**: `curl -X DELETE "https://[HOST]/api/resources/disk2/oldfile.txt" -H "X-Auth: [JWT_TOKEN]"`

### 6. Read File Content

**Endpoint**: `GET /api/resources/[PATH]`
**Output**: Returns JSON. The file content is in the `content` field.
**Example**: `curl -X GET "https://[HOST]/api/resources/disk2/notes.txt" -H "X-Auth: [JWT_TOKEN]"`

### 7. Update File Content (Overwrite)

**Endpoint**: `PUT /api/resources/[PATH]`
**Example**:

```bash
curl -X PUT "https://[HOST]/api/resources/disk2/notes.txt" \
  -H "X-Auth: [JWT_TOKEN]" \
  -H "Content-Type: text/plain" \
  --data "New file content goes here"
```

---

## Percent-Encoding Reference

Mandatory for `destination` parameters and paths with special characters.

| Char | Encoded | Char | Encoded | Char | Encoded |
| :--- | :------ | :--- | :------ | :--- | :------ |
| `/`  | `%2F`   | ` `  | `%20`   | `:`  | `%3A`   |
| `?`  | `%3F`   | `#`  | `%23`   | `&`  | `%26`   |
| `=`  | `%3D`   | `+`  | `%2B`   | `%`  | `%25`   |
| `[`  | `%5B`   | `]`  | `%5D`   | `@`  | `%40`   |

---

## Procedural Guidance for AI Models

1. **JSON Handling**: Always parse API responses. If an error occurs, report the HTTP status code and any error message in the JSON body.
2. **Path Safety**: Before `?` in URLs, do not encode slashes. Inside query parameters like `destination`, **always** encode slashes as `%2F`.
3. **Implicit Auth**: If a token is available from a previous turn, reuse it. If a request fails with 401 or 403, attempt to re-login.
4. **Interactive Formatting**: Present directory listings as Markdown tables for readability.
5. **Mandatory Confirmation**: **ALWAYS** ask for explicit user confirmation before deleting any file or folder. The confirmation prompt MUST include the full path of the item to be deleted.
