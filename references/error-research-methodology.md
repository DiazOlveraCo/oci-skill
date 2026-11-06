# Error Research Methodology

## Purpose

Avoid getting stuck in error loops by following a structured research process before retrying failed operations. This is a Layer 3 reference document — it describes the methodology in natural language. Implementation details belong in external tools.

---

## General Principle (GENERAL PRINCIPLE)

**Core Rule:** Before retrying a failed operation, you MUST research the error to understand its root cause. Blind retries without research are prohibited.

**Prohibition:** Do NOT run the same failing command more than twice without researching the error between attempts.

**Universal Scope:** All OCI operations, CLI commands, API calls, and infrastructure automation.

---

## Error Research Protocol

### Phase 1: Capture (Immediate)

When an error occurs:
1. **Record the exact error** — full message, error code, HTTP status, request ID
2. **Record the command** — exact command that failed, including all flags
3. **Record context** — instance state, auth method, region, recent changes
4. **Stop** — do not retry yet

### Phase 2: Classify (Within 30 seconds)

Categorize the error:

| Category | Examples | Action |
|----------|----------|--------|
| **Auth** | 401, NotAuthenticated, token expired | Refresh token, check config |
| **Capacity** | Out of host capacity, LimitExceeded | Wait, retry with backoff, try different AD |
| **State** | STOPPING, Conflict, InvalidState | Wait for state transition, check lifecycle |
| **Network** | Connection timeout, Connection refused | Check security lists, instance health, SSH daemon |
| **Config** | Invalid parameter, missing required field | Fix command syntax, check required params |
| **Service** | 5xx, InternalError, ServiceUnavailable | Wait, check OCI status page, contact support |

### Phase 3: Research (1-5 minutes)

Use available research tools:

1. **SearXNG MCP** — search for error message + "Oracle Cloud" + "OCI"
   - Query pattern: `"<error code>" OCI Oracle Cloud`
   - Query pattern: `"<error message>" site:docs.oracle.com`

2. **OCI Documentation** — check official docs
   - Troubleshooting guides
   - Known issues pages
   - API reference for error codes

3. **Session history** — check if this error occurred before
   - Use recursive skill to search past sessions
   - Look for similar error patterns

4. **Community knowledge** — Reddit, Stack Overflow, Oracle Communities
   - Search: `"OCI" "<error>" free tier`

### Phase 4: Decide (Based on Research)

| Research Outcome | Action |
|------------------|--------|
| Known issue with workaround | Apply workaround, then retry |
| Transient error (capacity, 5xx) | Exponential backoff retry (max 3) |
| Configuration error | Fix config, then retry |
| State mismatch | Wait for correct state, then retry |
| Unknown error | Escalate: create console connection, check logs |
| Repeated failure (3+ attempts) | STOP — escalate to human, try different approach |

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It Fails | Correct Approach |
|--------------|--------------|------------------|
| Retry same command 5+ times | No state change, wastes time | Research after 2nd failure |
| Ignore error code/message | Missing root cause info | Capture and classify first |
| Try random fixes | Unlikely to address root cause | Research → classify → targeted fix |
| Assume "it worked before" | Environment may have changed | Verify current state each time |
| Skip waiting for state transitions | OCI operations are async | Poll state until target reached |

---

## OCI-Specific Error Patterns

### Instance State Errors

| Error | Meaning | Resolution |
|-------|---------|------------|
| `Conflict` on START/STOP | Instance in transitional state | Poll `lifecycle-state` until RUNNING/STOPPED |
| `InvalidState` | Action not allowed in current state | Check valid state transitions in docs |
| `STOPPING` stuck >5 min | OCI agent issue | Use SOFTSTOP, then START; or console connection |

### SSH Errors

| Error | Meaning | Resolution |
|-------|---------|------------|
| `Connection refused` | sshd not listening | Serial console → `systemctl start sshd` |
| `Connection timed out during banner exchange` | sshd hung/starting | Wait 30-60s after RUNNING; serial console |
| `Permission denied (publickey)` | Key not authorized | Add key via console or cloud-init |

### Capacity Errors

| Error | Meaning | Resolution |
|-------|---------|------------|
| `Out of host capacity` | No free tier capacity in AD | Try different AD, different shape, or wait |
| `LimitExceeded` | Free tier quota exceeded | Check limits in console, delete unused resources |

---

## Research Tool Integration

### SearXNG MCP (Primary)
```bash
# Use via tools.sxn.query
# Category: "general" for technical errors
# Category: "it" for deeper technical articles
```

### Recursive Skill (Session History)
```bash
# Search past sessions for same error
# Use: "session_message.data LIKE '%error_code%'"
```

### Web Search (Fallback)
```bash
# tools.websearch for non-technical context
# Use when SearXNG returns insufficient results
```

---

## Decision Checklist

Before ANY retry, confirm:

- [ ] Error captured exactly (code, message, request ID)
- [ ] Error classified into category
- [ ] Research performed (SearXNG + docs + history)
- [ ] Root cause hypothesis formed
- [ ] Targeted fix identified (not random)
- [ ] State verified (instance RUNNING, auth valid, etc.)
- [ ] Max retries not exceeded (3 attempts max)

If ANY checkbox is NO → STOP → Research more → Do not retry.

---

## Escalation Path

When research doesn't yield a fix after 3 attempts:

1. **Create serial console connection** — direct VM access
2. **Check system logs** — `journalctl -u sshd`, `dmesg`
3. **Verify cloud-init** — `/var/log/cloud-init.log`
4. **Detach/attach boot volume** — repair offline
5. **Recreate instance** — last resort, use automation script

---

## Documentation Standard

When documenting an error resolution in skill references:

1. **Error signature** — exact code + message pattern
2. **Root cause** — what actually caused it
3. **Fix** — specific commands/config changes
4. **Verification** — how to confirm fix works
5. **Prevention** — config changes to avoid recurrence

Example entry in troubleshooting table:
| Symptom | Root Cause | Fix | Verification |
|---------|------------|-----|--------------|
| `Connection refused` on port 22 | `sshd.service` failed to start on boot (known OCI OL9 issue) | Serial console → `systemctl enable --now sshd`; add `haveged` for entropy | `ssh oci "echo OK"` succeeds |

---

## Notes

- This methodology applies to ALL OCI operations, not just SSH
- The goal is to reduce time-to-resolution by avoiding blind retries
- Research time (1-5 min) is always less than repeated failure time (10-30 min)
- When in doubt, STOP and research — never guess