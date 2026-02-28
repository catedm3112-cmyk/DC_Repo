# Connection Doctor: ChatGPT Apps + MCP (Shortest-Path Playbook)

Use this as a **surgical** runbook to resolve connection failures with minimal steps and minimal irreversible actions.

---

## 0) Intake (30 seconds)
Collect only:
- App/MCP name
- Exact error text (copy/paste) + screenshot
- Plan tier (Plus / Pro / Business / Enterprise)
- Connection type: standard App vs custom MCP server URL
- Failing action: read vs write

---

## 1) Shortest-Path Decision Tree (9 nodes)

1. **Classify connection type**
   - Standard App? → go to node 2
   - Custom MCP URL? → go to node 5

2. **Identify error family from exact text**
   - "access denied", "admin approval required", "not allowed" → node 3
   - "invalid_grant", "redirect_uri_mismatch", "state", "consent" → node 4
   - "unauthorized", "forbidden", "insufficient_scope", read works/write fails → node 8

3. **Permission/admin gate**
   - Business/Enterprise: ask workspace admin to approve/allow app + scopes.
   - Re-test read action first.
   - If still failing, go to node 4.

4. **OAuth repair (least destructive first)**
   1) ChatGPT **Settings → Apps → [App] → Reconnect**.
   2) Complete consent in provider login window with the intended account/tenant.
   3) If persistent OAuth mismatch errors, disconnect then reconnect once.
   4) Re-test read.

5. **Custom MCP availability check**
   - If custom MCP option missing/disabled in UI, likely plan/admin policy gate.
   - Enable policy/feature, then retry URL setup.
   - If available but failing, go to node 6.

6. **Transport + discovery sanity (custom MCP)**
   - MCP URL reachable over HTTPS.
   - `/.well-known/mcp` (or configured endpoint) returns valid server metadata.
   - No TLS/cert errors; no auth proxy rewriting headers.
   - If discovery fails, fix endpoint first.

7. **Spec compliance check (custom MCP)**
   - Validate required MCP fields and tool schema correctness.
   - Ensure at least one callable read tool exists.
   - Patch mismatches (see compliance checklist below), then reconnect.

8. **Scope/token drift (stale credentials)**
   - Symptom: read succeeds, write fails or newly added scopes ignored.
   - Reconnect app to refresh tokens and grant updated scopes.
   - Verify provider-side app scopes include write/draft permission.

9. **Final confirmation**
   - Run one harmless read + one harmless draft-only write.
   - If both pass, stop. If write fails only, return to node 8.

---

## 2) “Do This First” Checklist

1. Capture exact error text + screenshot.
2. Confirm whether this is standard App or custom MCP URL.
3. In ChatGPT, go to:
   - **Profile/Workspace menu → Settings → Apps**
   - Find the affected integration
   - Click **Reconnect** (first choice)
   - Only if needed: **Disconnect** then reconnect once
4. Re-run a **read** action (safer, fastest signal).
5. If read passes and write fails, treat as scope/permission and re-check consent scopes.

---

## 3) Custom MCP Compliance Checklist (OpenAI MCP Spec-Oriented)

### A) Server + transport basics
- [ ] HTTPS endpoint is reachable from public internet (no local-only hostnames).
- [ ] Valid TLS certificate chain.
- [ ] Discovery metadata endpoint responds with JSON, not HTML/auth redirect.
- [ ] Stable URL (no short-lived tunneling URLs in production).

### B) Required protocol structure
- [ ] MCP protocol version fields present and supported.
- [ ] Server identity metadata present (`name`, `version` where required).
- [ ] `tools/list` returns an array of tool descriptors.
- [ ] `tools/call` accepts declared inputs and returns structured output.

### C) Tool schema quality (most common breakage)
- [ ] Every tool has unique `name` and human-readable `description`.
- [ ] Input schema is valid JSON Schema object.
- [ ] `required` keys exactly match properties that are truly mandatory.
- [ ] No undeclared fields required at runtime.
- [ ] Enum/range constraints align with server-side validation.

### D) Auth expectations
- [ ] Server clearly indicates required auth mode (none/API key/OAuth token pass-through).
- [ ] 401 vs 403 errors are semantically correct.
- [ ] Token expiry handling returns actionable error messages.

### E) Safeguards for write actions
- [ ] Provide at least one **draft/simulate** write tool variant when possible.
- [ ] Idempotency key support for create/update operations.
- [ ] Explicit dry-run flag (`dry_run: true`) where feasible.

### Most common missing fields/issues and quick patches
1. **Missing/weak `description` on tools**
   - Patch: add concise behavior + side effects in description.
2. **Input schema not strict enough**
   - Patch: add `type: "object"`, `properties`, and `required`.
3. **Tool names unstable or duplicated**
   - Patch: enforce stable snake_case names and uniqueness.
4. **Write tool lacks safe mode**
   - Patch: add `dry_run` boolean defaulting to `true` for verification flows.
5. **Discovery endpoint returns auth HTML**
   - Patch: bypass SSO redirect for machine endpoint or use dedicated MCP host.

### Minimal compliant tool descriptor template
```json
{
  "name": "create_draft_message",
  "description": "Create a draft message without sending it.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "to": { "type": "string", "description": "Recipient email" },
      "subject": { "type": "string" },
      "body": { "type": "string" },
      "dry_run": { "type": "boolean", "default": true }
    },
    "required": ["to", "subject", "body"]
  }
}
```

---

## 4) Final Verification Script (Read + Write, Minimal Risk)

Run this sequence after reconnect/fix:

1. **Read probe (harmless)**
   - Example: “List my top 3 recent items/messages/tasks only.”
   - Pass criteria: successful response with expected object IDs/timestamps.

2. **Write probe (draft-only, no-send)**
   - Example: “Create a draft message/task titled `Connection test - do not send`.”
   - Pass criteria: returns draft ID/object ID, and provider UI shows draft/unpublished object.

3. **Cleanup**
   - Delete the test draft/object (optional but recommended).

4. **Record evidence**
   - Save: timestamp, tool/action name, returned ID, screenshot.

---

## 5) Triage Notes by Symptom
- **Read fails + write fails**: connectivity, OAuth, or admin policy first.
- **Read works + write fails**: almost always missing write scopes, stale token, or provider role limitation.
- **Intermittent failures**: rotating tunnel URL, flaky auth proxy, or token refresh race.

---

## 6) Minimal-Action Repair Order (default)
1. Reconnect in ChatGPT Settings → Apps.
2. Retry read probe.
3. If write fails, reconnect with explicit consent for write scope.
4. If custom MCP, validate discovery + tool schema.
5. Escalate admin/policy only if explicitly blocked by permission error text.

