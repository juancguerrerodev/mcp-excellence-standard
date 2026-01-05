# MCP Excellence Standard - Compliance Checklist

Use this checklist to evaluate any MCP implementation against the Excellence Standard.

---

## Instructions

1. Review each category
2. Check items that are implemented
3. Calculate score per category
4. Sum for total score
5. Compare against rating scale

---

## 1. Context Window Optimization (Score: __/6)

| # | Requirement | ✓ |
|---|-------------|---|
| 1.1 | `returnOnlyIds` mode on all list/search operations | ☐ |
| 1.2 | `compact` mode on all list/search operations | ☐ |
| 1.3 | Field selection support (`fields` parameter) | ☐ |
| 1.4 | Cursor-based pagination (`pageToken`) | ☐ |
| 1.5 | Dedicated `count_*` operations for each resource | ☐ |
| 1.6 | Batch operations return summaries only (not item lists) | ☐ |

**Notes:**
- Default limits should be conservative (10-25, not 100-500)
- `compact` should truncate subjects/bodies, exclude heavy fields

---

## 2. Atomic vs Composite Operations (Score: __/5)

| # | Requirement | ✓ |
|---|-------------|---|
| 2.1 | `search_and_action` composite for list + modify workflows | ☐ |
| 2.2 | `get_or_create` pattern for creatable resources | ☐ |
| 2.3 | `dryRun` parameter on all destructive/bulk operations | ☐ |
| 2.4 | Internal batching in bulk operations (not exposed to agent) | ☐ |
| 2.5 | Upsert semantics where API supports it | ☐ |

**Notes:**
- Composites dramatically reduce token consumption
- `dryRun` allows agent to preview and confirm with user

---

## 3. Error Handling & Resilience (Score: __/5)

| # | Requirement | ✓ |
|---|-------------|---|
| 3.1 | Batch operations continue after individual failures | ☐ |
| 3.2 | All errors include code, message, and suggestion | ☐ |
| 3.3 | Internal retry for transient/rate-limit errors | ☐ |
| 3.4 | Partial results returned with warnings when possible | ☐ |
| 3.5 | Rate limit info exposed in responses | ☐ |

**Notes:**
- Errors should be machine-parseable
- Never let one bad item kill an entire batch

---

## 4. Authentication & Multi-Account (Score: __/5)

| # | Requirement | ✓ |
|---|-------------|---|
| 4.1 | All operations accept `accountId` parameter | ☐ |
| 4.2 | `list_accounts()` tool available | ☐ |
| 4.3 | Automatic token refresh (no agent intervention) | ☐ |
| 4.4 | `get_auth_status()` for checking auth state | ☐ |
| 4.5 | Credentials stored in system keychain/secure storage | ☐ |

**Notes:**
- Token refresh should be invisible to the agent
- Device flow support is bonus (not scored)

---

## 5. Discovery & Introspection (Score: __/4)

| # | Requirement | ✓ |
|---|-------------|---|
| 5.1 | `get_capabilities()` tool available | ☐ |
| 5.2 | List operations return enum/option values | ☐ |
| 5.3 | Tool descriptions include examples | ☐ |
| 5.4 | Schema introspection available | ☐ |

**Notes:**
- Agent should be able to discover what's possible programmatically
- Links to documentation in descriptions is helpful

---

## 6. State & Caching (Score: __/4)

| # | Requirement | ✓ |
|---|-------------|---|
| 6.1 | ETag support for conditional requests | ☐ |
| 6.2 | `since` / `after` parameters for incremental sync | ☐ |
| 6.3 | Internal caching for stable data (labels, folders) | ☐ |
| 6.4 | `invalidate_cache()` tool available | ☐ |

**Notes:**
- These reduce redundant API calls
- Watch/subscribe is bonus if platform supports

---

## 7. Resource Efficiency (Score: __/5)

| # | Requirement | ✓ |
|---|-------------|---|
| 7.1 | `expand` parameter for nested resources | ☐ |
| 7.2 | Heavy fields excluded by default (body, attachments) | ☐ |
| 7.3 | `includeBody` / `includeContent` opt-in flags | ☐ |
| 7.4 | Truncation with metadata (`truncated: true`, `totalLength`) | ☐ |
| 7.5 | Attachment metadata separate from content download | ☐ |

**Notes:**
- Never auto-download attachments
- Default to minimal data

---

## 8. Semantic Grouping (Score: __/4)

| # | Requirement | ✓ |
|---|-------------|---|
| 8.1 | Consistent `{resource}_{action}` naming | ☐ |
| 8.2 | Composite tools for common workflows | ☐ |
| 8.3 | Clear danger/destructive indicators (⚠️) | ☐ |
| 8.4 | Logical grouping (read/write/delete separation) | ☐ |

**Notes:**
- Naming consistency helps agent select correct tool
- Dangerous operations should be clearly marked

---

## 9. Audit & Observability (Score: __/4)

| # | Requirement | ✓ |
|---|-------------|---|
| 9.1 | `get_action_history()` available | ☐ |
| 9.2 | Undo support for reversible operations | ☐ |
| 9.3 | Change summaries in modification responses | ☐ |
| 9.4 | `dryRun` mode on all bulk/destructive operations | ☐ |

**Notes:**
- Change summaries show before/after state
- Undo improves recoverability

---

## 10. Agent-Friendly Output (Score: __/5)

| # | Requirement | ✓ |
|---|-------------|---|
| 10.1 | All responses are JSON objects (never formatted text) | ☐ |
| 10.2 | Consistent field names across all tools | ☐ |
| 10.3 | All created resources return usable IDs | ☐ |
| 10.4 | Status values are enums, not text | ☐ |
| 10.5 | Timestamps in ISO 8601 format | ☐ |

**Notes:**
- Same resource should have identical schema everywhere
- IDs must be immediately usable in subsequent calls

---

## 11. Safety & Guardrails (Score: __/6)

| # | Requirement | ✓ |
|---|-------------|---|
| 11.1 | Maximum batch sizes enforced (hard cap) | ☐ |
| 11.2 | Two-stage deletion (trash → delete) | ☐ |
| 11.3 | Confirmation tokens for bulk destructive operations | ☐ |
| 11.4 | Rate limiting on mutations | ☐ |
| 11.5 | Read-only mode support | ☐ |
| 11.6 | Scope/permission restrictions configurable | ☐ |

**Notes:**
- Safety is non-negotiable
- One bad agent decision shouldn't cause data loss

---

## Scoring Summary

| Category | Points | Score |
|----------|--------|-------|
| 1. Context Window | /6 | __ |
| 2. Composite Operations | /5 | __ |
| 3. Error Handling | /5 | __ |
| 4. Authentication | /5 | __ |
| 5. Discovery | /4 | __ |
| 6. State & Caching | /4 | __ |
| 7. Resource Efficiency | /5 | __ |
| 8. Semantic Grouping | /4 | __ |
| 9. Audit | /4 | __ |
| 10. Output | /5 | __ |
| 11. Safety | /6 | __ |
| **TOTAL** | **/53** | **__** |

---

## Rating Scale

| Score | Rating | Recommendation |
|-------|--------|----------------|
| 45-53 | ⭐⭐⭐⭐⭐ Excellent | Production ready |
| 36-44 | ⭐⭐⭐⭐ Good | Minor improvements needed |
| 27-35 | ⭐⭐⭐ Adequate | Significant gaps to address |
| 18-26 | ⭐⭐ Poor | Major refactoring recommended |
| 0-17 | ⭐ Failing | Consider rebuilding |

---

## Priority Order for Improvements

If scoring low, prioritize in this order:

1. **Context Window Optimization** - Highest ROI, immediate token savings
2. **Safety & Guardrails** - Prevent catastrophic mistakes
3. **Composite Operations** - Reduce API round-trips
4. **Error Handling** - Make failures actionable
5. **Output Format** - Consistent, machine-parseable
6. **Discovery** - Help agent understand capabilities
7. **Everything else** - Nice to have

---

## Quick Evaluation Questions

For rapid assessment, answer these:

1. Can I get just IDs without full objects? (Context Window)
2. Can I preview before executing? (dryRun)
3. What happens if one item in a batch fails? (Error Handling)
4. Can I accidentally delete everything? (Safety)
5. Is the output JSON or formatted text? (Output)

If any answer is problematic, the MCP needs work.

---

*Last updated: January 2026*
*MCP Excellence Standard v1.0*
