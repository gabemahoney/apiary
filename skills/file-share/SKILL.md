---
name: file-share
description: >
  Pastebin-style file sharing via the Omniverse Storage API. Upload files or
  text under unguessable addresses and share them out-of-band.
triggers:
  - share a file
  - upload a file
  - download a file
  - send someone a file
  - pass data to another agent
  - share a snippet
  - file-share
tools:
  - file-share (CLI)
dependencies:
  - python >=3.10
  - httpx
---

# File Share Skill

Pastebin-like file sharing via the Omniverse Storage API. Upload files or text under unguessable addresses and share the address out-of-band. No listing or browsing -- if you have the address, you can download.

## When to Use

Use this skill when:
- You need to share a file, snippet, or directory with a colleague or another agent
- You need to pass data between agents via a shared storage layer
- You want to upload artifacts (logs, configs, CSVs, images) for someone to download later

The address is the sharing mechanism. Hand it out via chat, email, or any other channel.

## Prerequisites

- Python 3.10+
- `pip` or `uv` available
- VPN or network access to the Storage API discovery endpoint

## Setup (first time only)

Install the CLI tool from the skill directory:

```bash
pip install -e /path/to/file-share-skill
```

Or with uv:

```bash
uv pip install -e /path/to/file-share-skill
```

After install, the `file-share` command is available globally.

### Pick a storage location (first time only -- remembered after)

```bash
file-share locations
file-share set-location <address>
```

## Quick Reference

### Upload a file

```bash
file-share upload ~/reports/q4.csv
file-share upload ~/photos/diagram.png
```

Prints an unguessable address like `njohns/Kx7mR2pQ9vN1bL3w.csv`. The original file extension is preserved; the name is replaced with a cryptographic token.

### Upload a directory

```bash
file-share upload ~/project/results/
```

The directory is packed into an uncompressed tar archive before upload. The output includes the file count and archive size.

### Download a file

```bash
file-share download "njohns/Kx7mR2pQ9vN1bL3w.csv" ~/Downloads/q4.csv
```

Downloads are streamed to disk -- no file size limit on memory.

### Download and extract a directory

```bash
file-share download --extract "njohns/Ax9bR2pQ.tar" ~/results/
```

The `--extract` flag treats the remote file as a tar archive and extracts it into the destination directory.

### Share text content

```bash
file-share share "meeting notes here"
file-share share --ext .yaml "key: value"
echo "piped content" | file-share share -
```

### Read a small text file

```bash
file-share read "njohns/aB3xZ9qW.yaml"
```

For binary files or large content, use `download` instead.

### Delete a file

```bash
file-share delete "njohns/Kx7mR2pQ9vN1bL3w.csv"
```

## Authentication

On first use, the CLI authenticates via NVIDIA SSO (OpenID Connect over REST):
- If a `client_secret` is available, it uses the **device code flow** (prints a URL + code)
- Otherwise, it opens a **browser** for PKCE login

The token is cached and reused automatically. On Unix the cache directory is `~/.omni-file-share/`; on Windows it is `%LOCALAPPDATA%\omni-file-share\`. Long-running operations refresh the token before expiry. No credentials are stored.

## Address Model

Every upload generates an unguessable address: `{username}/{token}{ext}`. The token is 22 characters from `secrets.token_urlsafe(16)` (128 bits of entropy). There is no way to list or guess other users' files -- sharing happens out-of-band.

Addresses support shorthand:
- `Kx7mR2pQ9vN1bL3w.csv` -- resolves to your namespace
- `alice/aB3xZ9qW.tar` -- another user's file
- Full URI -- passed through as-is

## Upload Strategy

The CLI queries the server for recommended upload thresholds and selects the best method automatically:

| Method | How it works |
|--------|-------------|
| Body upload | Entire file in PUT request body (small files) |
| Redirect upload | Streams to a pre-signed URL returned by the server |
| Multipart upload | Chunked upload with per-part pre-signed URLs, abort on failure |

All uploads use streaming I/O -- only one chunk is in memory at a time. The `Expect: 100-continue` header is sent per the API spec.

## Environment Variables (optional)

| Variable | Description |
|---|---|
| `STORAGE_DISCOVERY_URL` | Discovery endpoint (default: `https://pastebin.storage.omni.nvidia.com`) |
| `STORAGE_AUTH_TOKEN` | Static bearer token (skips SSO) |
| `STORAGE_USERNAME` | Override username (default: system username) |
| `STORAGE_RESOURCE_ADDRESS` | Pre-set storage location (skips set-location step) |

## Agent Usage Notes

- After install, `file-share` works from any directory.
- The `share` command accepts `-` as content to read from stdin (useful for piping).
- Binary files cannot be read with `read` -- use `download` instead.
- All I/O is async and streaming. Uploads and downloads of any size use constant memory.
- For large transfers, run in a background shell or subagent to avoid blocking the conversation.
- Transient failures (network blips, 429, 5xx) are retried automatically with exponential backoff.

## Files

| File | Purpose |
|---|---|
| `pyproject.toml` | Package definition and `file-share` CLI entry point |
| `requirements.txt` | Python dependencies (httpx) |
| `src/file_share/cli.py` | CLI entry point with all subcommands |
| `src/file_share/auth.py` | OAuth authentication (device code + browser PKCE) |
| `src/file_share/config.py` | Configuration from env vars + cached preferences |
| `src/file_share/storage.py` | Async streaming HTTP client for the Storage API |
