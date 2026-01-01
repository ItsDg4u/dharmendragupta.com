---
title: 🎉 Intercepting Claude Desktop MCP Traffic with Burp Suite
subtitle: "A pentester's guide to debugging remote Model Context Protocol servers by forcing Claude to talk through a local proxy."
summary: Learn how to force Claude Desktop to route traffic through Burp Suite for full visibility into MCP JSON-RPC messages, including fixes for HTTP/2 and SSE buffering issues
date: 2026-01-01
diagram: true

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
# image:
#   caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - MCP
  - Claude
  - Burp Suite
  - Debugging
  - Security
---

<!-- Welcome 👋 -->

{{< toc mobile_only=true is_open=true >}}

## Overview
If you are pentesting [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) servers, the **Claude Desktop** app is the ultimate integration test. But debugging it can feel like working in a black box.

When your connection fails or tools don't show up, you often get a generic *"Connection Failed"* message. There are no detailed logs, no network tab, and no visibility into the JSON-RPC messages flying back and forth.

I recently needed to connect Claude Desktop to a remote MCP server while routing the traffic through **Burp Suite** to inspect the payloads. It turned out to be much harder than just setting an environment variable.

After hours of debugging Node.js timeouts, HTTP/2 protocol errors, and silent crashes, I found the working configuration. Here is the complete guide to gaining full visibility into your MCP traffic.

## The Architecture
Before we fix the config, it helps to understand *why* this is difficult. We are trying to force a strictly typed Node.js client (inside Claude) to talk through a Java-based proxy (Burp) to a remote server, all while maintaining a persistent Server-Sent Events (SSE) stream.

```mermaid
sequenceDiagram
    autonumber
    participant Claude as Claude Desktop App
    participant Shell as Bash Shell<br/>(via config args)
    participant Burp as Burp Proxy<br/>(127.0.0.1:8080)
    participant Remote as Remote MCP Server

    Note over Claude, Shell: 1. Claude launches the command defined in JSON
    Claude->>Shell: Spawns process with env vars
    
    Note right of Shell: 2. Process starts
    Shell->>Burp: HTTPS Connect (Tunnel)
    
    Note right of Burp: 3. Burp intercepts traffic<br/>(HTTP/2 Disabled)
    Burp->>Remote: POST /mcp (JSON-RPC)
    
    Note right of Remote: 4. Server streams response<br/>(SSE / Chunked)
    Remote-->>Burp: 200 OK (Tools List)
    
    Note left of Burp: 5. Burp streams instantly
    Burp-->>Shell: Forward Response
    Shell-->>Claude: STDOUT (JSON)

    Note over Claude: 6. Connector Switch turns Green
```

### Step 1: The "All-in-One" Configuration
You don't need complex wrapper scripts. You can achieve full proxy interception directly inside your claude_desktop_config.json file.

First, install the mcp-remote tool globally to avoid npx timeouts:

```bash
npm install -g mcp-remote
```

Then, use this configuration (adjust paths for your OS):
claude_desktop_config.json
```
{
  "mcpServers": {
    "server": {
      "command": "/bin/bash",
      "args": [
        "-c",
        "/opt/homebrew/bin/mcp-remote https://your-remote-server.com/mcp --enable-proxy --header \"Authorization: Bearer $API_TOKEN\" 2> /tmp/mcp_stderr.log"
      ],
      "env": {
        "API_TOKEN": "YOUR_ACTUAL_TOKEN_HERE",
        "HTTP_PROXY": "http://127.0.0.1:8080",
        "HTTPS_PROXY": "http://127.0.0.1:8080",
        "NODE_TLS_REJECT_UNAUTHORIZED": "0",
        "PATH": "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
      }
    }
  }
}
```

💡**Why This Works (The "Gotchas")**

* **Absolute Paths:** GUI apps don't load your $PATH. You must point directly to the binary (e.g., /opt/homebrew/bin/mcp-remote).
* **Proxy Flags:** The mcp-remote tool ignores proxy env vars unless you explicitly pass --enable-proxy.
* **Logging:** Redirecting stderr to a file (2> /tmp/mcp_stderr.log) gives you the only visibility into why a connection dies. Never redirect stdout, or you will break the pipe to Claude.

### Step 2: Configuring Burp Suite (Critical)
Node.js's HTTP client (undici) is extremely strict. Default Burp settings will cause the connection to crash immediately. You must apply these three specific fixes.

#### Fix A: Disable HTTP/2 (Prevent Protocol Crashes)

Burp often tries to downgrade HTTP/2 connections incorrectly, causing "Invalid Header" errors. You must disable it on **both** sides.

* **Listener:** Go to **Proxy** > **Proxy Settings** > **Edit Proxy Listeners (127.0.0.1:8080)** > **HTTP Tab** > **Uncheck "Support HTTP/2"**.
![](./Listener.png)
* **Upstream:** Go to **Settings** > **Network** > **HTTP** > **HTTP/2** > **Uncheck "Enable HTTP/2"**.
![](./Upstream.png)

#### Fix B: Prevent Connection Blocking (The "One Lane" Fix)

Since we disabled HTTP/2, the initial SSE stream "hogs" the single TCP connection. If you try to run a tool, the request will hang because the socket is busy. We must force a new connection for every request.

1.  Go to: **Proxy** > **Match and Replace**.
2.  **Add Rule:**
    * **Type:** Request header
    * **Match:** Connection: keep-alive
    * **Replace:** Connection: close
    ![](./Match_Replace.png)

#### Fix C: Enable Streaming Responses

Tell Burp explicitly not to wait for this URL.

1.  Go to: **Settings** > **Network** > **HTTP** > **Streaming Responses**.
2.  **Add URL:** your-server-domain.com
3.  **Crucial:** Check **"Store streaming responses"** (so you can see them) and Uncheck **"Strip chunked encoding metadata"**.
![](./Streaming_Responses.png)

### Step 3: The "Toggle Trick" (Pro Tip)

> If you have done all the above and Claude still times out during the initial connection (showing a disabled switch), use the **Pass-Through Toggle Trick**.

The initial handshake (**tools/list**) is often too large and fragile for the proxy to buffer instantly.

1.  **Enable Pass Through:** Go to **Settings** > **Tools** > **Proxy** > **TLS Pass Through** and add your server domain.
2.  **Start Claude:** The connection will succeed immediately (bypassing Burp) and the switch will turn Green.
3.  **Disable Pass Through:** While Claude is running, **remove** the domain from the TLS Pass Through list.
![](./Toggle_Trick.png)

📊**Result:** The connection stays alive, but now Burp will begin capturing all *subsequent* traffic (like tool executions and prompts) perfectly.

## 🚀 Conclusion

Debugging MCP servers doesn't have to be a guessing game. By forcing the traffic through a proxy, you can inspect every tool call, fuzz inputs, and verify security controls—all from within your local development flow.

Happy Intercepting!


<style>
  .mermaid {
    background-color: white !important;
    padding: 20px;
    border-radius: 8px;
  }
</style>