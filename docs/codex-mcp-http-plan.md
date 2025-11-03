# Plan: Expose `codex mcp-server` via Streamable HTTP using `mcp-streamablehttp-proxy`

1. **Confirm prerequisites**
   - Ensure Codex CLI is installed and accessible on the PATH (`codex --version`).
   - Verify that Anaconda (or Miniconda) is available (`conda --version`); install it if missing.

2. **Create the dedicated Conda environment**
   - Run `conda create -n codex-mcp-proxy python=3.11` to provision an isolated environment.
   - Activate it with `conda activate codex-mcp-proxy`.

3. **Install the proxy package inside the environment**
   - With the environment active, install the bridge from the fork with Codex fixes:
     ```
     pip install --upgrade git+https://github.com/Chargeuk/mcp-streamablehttp-proxy.git
     ```
   - Optionally pin the package version in `requirements.txt` if we want reproducibility.

4. **Wrap the Codex MCP server with the proxy**
   - Launch the bridge from the activated environment:
     ```
     mcp-streamablehttp-proxy --log-level debug --timeout 900 --host 127.0.0.1 --port 3022 codex mcp-server
     ```
   - The command starts `codex mcp-server` as a subprocess and exposes a streamable HTTP endpoint at `http://127.0.0.1:3022/mcp`.

5. **Validate the HTTP endpoint**
   - Initialize a session and capture the response headers so you can reuse the session ID:
     ```
     curl --max-time 60 -X POST http://127.0.0.1:3022/mcp \
       -H "Content-Type: application/json" \
       -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"codex-cli","version":"0.53.0"}}}' \
       -D /tmp/codex-mcp-headers.txt
     ```
   - Grab the `Mcp-Session-Id` value (for example `480c3137-6b70-47fa-a671-b26cb6f006d5`) and confirm the tools list:
     ```
     SESSION_ID=$(grep -i 'mcp-session-id' /tmp/codex-mcp-headers.txt | awk '{print $2}' | tr -d '\r')
     curl --max-time 60 -X POST http://127.0.0.1:3022/mcp \
       -H "Content-Type: application/json" \
       -H "Mcp-Session-Id: ${SESSION_ID}" \
       -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'
     ```
   - Use the same session header for any subsequent `tools/call` requests; if one times out, re-initialize to get a fresh session ID. Example call:
     ```
     curl --max-time 60 -X POST http://127.0.0.1:3022/mcp \
       -H "Content-Type: application/json" \
       -H "Mcp-Session-Id: ${SESSION_ID}" \
       -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"codex","arguments":{"prompt":"Are you ready to code?"}}}'
     ```
     Expected reply:
     `{"id":3,"jsonrpc":"2.0","result":{"content":[{"text":"Ready when you are—just let me know what you’d like to build.","type":"text"}]}}`

6. **Register the streamable HTTP server with Codex**
   - Update `~/.codex/config.toml` with:
     ```
     [mcp_servers.codex_http_proxy]
     url = "http://127.0.0.1:3022/mcp"
     tool_timeout_sec = 600.0
     startup_timeout_sec = 120.0
     ```
   - Restart the Codex CLI or IDE so it picks up the new configuration.

7. **Document operational guidance**
   - Capture start/stop procedures for the proxy.
   - Note any port, timeout, or security considerations (e.g., keep proxy bound to localhost unless behind an authenticated gateway).

8. **Maintain the environment**
   - Keep the `codex-mcp-proxy` environment updated (`conda activate codex-mcp-proxy && pip install --upgrade mcp-streamablehttp-proxy`).
   - Record troubleshooting steps (checking `~/.codex/log/codex-tui.log`, verifying proxy logs, etc.).
