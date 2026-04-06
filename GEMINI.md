# DevSpec Autopilot — MCP Tool Reference

## What is DevSpec?

DevSpec is an AI-native project planning and code intelligence platform. Teams use DevSpec to create, prioritize, and track action items (bugs, features, improvements, tasks). The autopilot system enables AI agents to autonomously pick up queued action items, implement them, and report results back to the team.

## What This Extension Does

This extension connects Gemini CLI to DevSpec's autopilot system. When you run `/autopilot:process`, Gemini fetches the next queued action item, claims it, implements the required changes, and reports completion with full audit details.

## MCP Tools

All tools are called via the `devspec` MCP server configured in `gemini-extension.json`. Gemini handles the MCP transport — you invoke tools by name.

---

### get_project_summary

Get project settings including autopilot configuration. Call this at startup to determine push/merge behavior.

**Parameters**: None (uses token-scoped project)

**Response** (relevant fields):
```json
{
  "local_plugin_settings": {
    "auto_push": true,
    "auto_merge": true,
    "target_branch": "staging",
    "branch_prefix": "work/action-item-",
    "custom_instructions": "..."
  }
}
```

**Key settings**:
- `auto_push`: Whether to push feature branches to remote
- `auto_merge`: Whether to merge feature branches into target_branch after push
- `target_branch`: Branch to merge into (e.g., "staging", "main")
- `branch_prefix`: Prefix for feature branches (e.g., "work/action-item-")
- `custom_instructions`: Additional instructions to follow during processing

---

### get_next_work_item

Fetch the next highest-priority queued action item with full context.

**Parameters**: None

**Success Response**:
```json
{
  "id": "4d753b6a-28d9-4d2d-8f23-af3103af76db",
  "title": "Fix login timeout on mobile",
  "description": "## Context\nThe login flow times out after 5 seconds on cellular connections...",
  "priority": "high",
  "type": "bug",
  "status": "open",
  "agent_status": "queued",
  "tags": ["auth", "mobile"],
  "affected_files": ["src/auth/login.ts", "src/auth/timeout.ts"],
  "parent_action_item_id": null
}
```

**Empty Queue**: Returns empty/null response — no items available.

**Errors**:
- `401 Unauthorized` — Token invalid or expired. Regenerate in DevSpec UI: Settings > Autopilot.
- `500 Internal Server Error` — Server-side failure. Retry once.

---

### claim_work_item

Atomically mark an action item as in_progress. Only one runner can claim a given item — this prevents race conditions when multiple runners are active.

**Parameters**:
```json
{
  "action_item_id": "4d753b6a-28d9-4d2d-8f23-af3103af76db"
}
```

**Success**: Item transitions from `queued` to `in_progress`.

**Errors**:
- `409 Conflict` — Another runner already claimed this item. This is a normal race condition. Print a message and exit — run again to get the next item.
- `401 Unauthorized` — Token invalid.
- `404 Not Found` — Item does not exist.

---

### complete_work_item

Mark an action item as completed with a full report. **All fields become part of the action item's permanent record visible to the entire team.** Write substantive content, not placeholders.

**Parameters**:
```json
{
  "action_item_id": "4d753b6a-28d9-4d2d-8f23-af3103af76db",
  "commit_sha": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
  "affected_files": [
    "src/auth/login.ts",
    "src/auth/timeout.ts",
    "tests/auth/login.test.ts"
  ],
  "completion_note": "## Changes\n\nIncreased the login timeout from 5 seconds to 30 seconds for mobile clients. The previous timeout was too aggressive for cellular connections where DNS resolution alone can take 3-5 seconds.\n\n## Approach\n\nModified the `createAuthClient` factory to accept a platform-specific timeout configuration. Mobile clients now use a dedicated timeout profile that accounts for higher-latency networks. Added a unit test verifying the mobile timeout is applied correctly.\n\n## Decisions\n\nChose to make timeout configurable per-platform rather than a global increase, since desktop clients benefit from the shorter timeout for faster failure detection.",
  "completion_summary": "Fixed login timeout on mobile by increasing the timeout from 5s to 30s for cellular connections. Mobile clients now use a platform-specific timeout profile that accounts for high-latency networks.",
  "testing_notes": "1. Open the app on a mobile device or mobile emulator\n2. Ensure the device is on a cellular connection (not WiFi)\n3. Attempt to log in with valid credentials\n4. Verify login completes successfully without timeout\n5. Verify desktop login still uses the original 5s timeout",
  "verification_report": {
    "verification_type": "partial",
    "automated_checks_passed": ["typecheck", "unit-tests"],
    "confidence": 0.85,
    "human_review_needed": ["Manual mobile device testing on cellular network"]
  }
}
```

**Field Quality Requirements**:

| Field | Requirements |
|-------|-------------|
| `commit_sha` | The actual git commit SHA. Required. |
| `affected_files` | Array of file paths that were changed. |
| `completion_note` | **Multi-paragraph markdown.** Must describe: what changed, why, and any decisions made. 2-3 paragraphs minimum. Do NOT write generic placeholder text. |
| `completion_summary` | **2-4 sentences.** User-facing changelog entry suitable for non-technical stakeholders. |
| `testing_notes` | **Numbered steps.** At least 3 steps describing how to verify the change works correctly. |
| `verification_report.verification_type` | One of: `automated` (all checks passed), `human_required` (needs manual review), `partial` (some automated, some manual). |
| `verification_report.automated_checks_passed` | Array of check names that passed (e.g., `["typecheck", "unit-tests", "lint"]`). |
| `verification_report.confidence` | Float 0.0–1.0. How confident you are the change is correct. |
| `verification_report.human_review_needed` | Array of areas needing manual review (optional). |

**Errors**:
- `401 Unauthorized` — Token invalid.
- `404 Not Found` — Item not found or not in `in_progress` state.
- `422 Unprocessable Entity` — Missing required fields.

---

### fail_work_item

Mark an action item as failed. Use this when you cannot complete the work — merge conflicts, persistent test failures, missing dependencies, etc.

**Parameters**:
```json
{
  "action_item_id": "4d753b6a-28d9-4d2d-8f23-af3103af76db",
  "error": "Merge conflict in src/auth/login.ts — upstream changes conflict with the timeout modification. Manual resolution required.",
  "partial_work": "Branch autopilot/action-item-4d753b6a pushed with partial changes. Timeout configuration added but not integrated due to merge conflict."
}
```

| Field | Requirements |
|-------|-------------|
| `error` | Clear description of what went wrong. Required. |
| `partial_work` | Notes on any work completed before failure — branch names, partial commits, etc. Optional but helpful. |

**Errors**:
- `401 Unauthorized` — Token invalid.
- `404 Not Found` — Item not found.

---

### send_heartbeat

Report runner presence to the DevSpec dashboard. Call after processing (success or failure). This is best-effort — if it fails, do not block the overall workflow.

**Parameters**:
```json
{
  "status": "idle",
  "runner_type": "ephemeral",
  "runner_hostname": "brandons-macbook.local",
  "cycle_count": 1,
  "tasks_completed": 1,
  "capabilities": ["code-generation", "git", "testing"]
}
```

| Field | Requirements |
|-------|-------------|
| `status` | `"working"` during processing, `"idle"` after completion. |
| `runner_type` | Always `"ephemeral"` for Gemini CLI runners. |
| `runner_hostname` | Auto-detect from system hostname. |
| `cycle_count` | Number of processing cycles in this session (usually `1`). |
| `tasks_completed` | Count of items completed successfully (0 or 1). |
| `capabilities` | `["code-generation", "git", "testing"]` |

**Errors**:
- `401 Unauthorized` — Token invalid.

---

## Error Handling Patterns

| Scenario | What To Do |
|----------|-----------|
| **Empty queue** (get_next_work_item returns null) | Print "No items queued — nothing to process." and exit cleanly. |
| **401 Unauthorized** (any tool) | Print error: "Authentication failed. Regenerate your token in DevSpec: Settings > Autopilot." Exit. |
| **409 Conflict** (claim_work_item) | Print: "Item already claimed by another runner. Run again to pick up the next item." Exit cleanly. |
| **API timeout** (any tool) | If item was claimed, call `fail_work_item` with timeout details. Otherwise, exit with timeout error. |
| **Test failures** (during implementation) | Attempt to fix once. If still failing, call `fail_work_item` with test failure details. |
| **Merge conflicts** (during git push) | Call `fail_work_item` with conflict file list and branch name for manual resolution. |
| **500 Server Error** (any tool) | Retry once. If still failing, treat as timeout scenario above. |
