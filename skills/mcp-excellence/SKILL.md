---
name: mcp-excellence
description: Build AI-agent-optimized Model Context Protocol (MCP) servers following the MCP Excellence Standard. Use when creating new MCPs, enhancing existing MCPs, wrapping APIs as MCPs, or when the user mentions MCP, Model Context Protocol, or wants to build tools for AI agents. This skill ensures MCPs are token-efficient (returnOnlyIds, compact mode, pagination), include composite operations (search_and_action, get_or_create), handle errors properly (partial success, structured errors), and implement safety guardrails (dryRun, confirmation tokens, batch limits).
---

# MCP Excellence Skill

Build Model Context Protocol servers optimized for AI agent consumption following the MCP Excellence Standard.

## Core Principle

**An MCP is NOT a human API.** It's an interface for an AI agent with:
- Limited context window (tokens are expensive)
- No persistent memory between calls
- Pattern-matching decision making
- Tendency toward verbose exploration without guardrails

## Quick Reference

For detailed patterns, read `references/patterns.md`. For platform-specific requirements, read `references/platforms.md`.

## Essential Patterns

### 1. Context Window Optimization (CRITICAL)

Every list/search operation MUST support:
- `returnOnlyIds: boolean` - Returns `{ count, ids[] }` only
- `compact: boolean` - Minimal fields, truncated content
- `pageSize: number` - Default 25, Max 100
- `pageToken: string` - Cursor pagination

Always add dedicated count tools:
```typescript
tools.add("count_{resource}");  // Returns { count: number }
tools.add("collect_{resource}_ids");  // Returns { count, ids[] }
```

Batch responses return summaries only:
```typescript
// ✅ CORRECT: { requested: 1000, deleted: 997, failed: 3 }
// ❌ WRONG: { deleted: ["id1", "id2", ...1000 items] }
```

### 2. Composite Operations

Reduce round-trips with composites:
```typescript
tools.add("search_and_action", {
  input: {
    query: string,
    action: "archive" | "trash" | "delete" | "markRead",
    dryRun?: boolean  // ALWAYS support dry run
  }
});

tools.add("get_or_create_{resource}");  // Idempotent creation
```

### 3. Error Handling

```typescript
interface MCPError {
  code: string;        // "LABEL_NOT_FOUND"
  message: string;     // Human-readable
  suggestion?: string; // "Use list_labels() to see available labels"
  recoverable: boolean;
}

// Batch operations continue after failures
interface BatchResult {
  success: number;
  failed: number;
  errors?: Array<{ id: string; error: string }>;
}
```

### 4. Safety & Guardrails

```typescript
// Hard limits on batch sizes
const MAX_BATCH_SIZE = 1000;

// Two-stage deletion
tools.add("trash_{resource}");   // Stage 1: Soft delete
tools.add("delete_{resource}");  // Stage 2: Only if trashed

// Confirmation tokens for dangerous operations
tools.add("bulk_delete", {
  input: { confirmToken?: string }
  // Without token: Returns { confirmationRequired, willDelete, confirmToken }
  // With token: Executes deletion
});

// Dry run on ALL destructive operations
{ input: { dryRun?: boolean } }
```

### 5. Output Format

```typescript
// ✅ JSON always, never formatted text
// ✅ Consistent schemas across all tools
// ✅ Return actionable IDs: { id: "Label_123", created: true }
// ✅ Status enums: { status: "sent" } not { status: "Email sent successfully" }
// ✅ ISO 8601 timestamps always
```

### 6. Tool Naming Convention

Use `{resource}_{action}` pattern:
```
email_list, email_get, email_send, email_delete
batch_move_emails, batch_delete_emails
email_search_and_action, email_analyze_inbox
get_capabilities, list_labels
```

## MCP Template Structure

```
my-mcp/
├── src/
│   ├── index.ts          # Entry point
│   ├── tools/
│   │   ├── read.ts       # List, get, search, count
│   │   ├── write.ts      # Create, update, send
│   │   ├── delete.ts     # Trash, delete
│   │   ├── batch.ts      # Batch operations
│   │   ├── composite.ts  # Search-and-action
│   │   └── discovery.ts  # Capabilities, schema
│   ├── auth/
│   │   └── oauth.ts      # Auth with token refresh
│   └── utils/
│       ├── errors.ts     # Error formatting
│       └── pagination.ts # Cursor handling
├── package.json
└── README.md
```

## Compliance Checklist

Before delivering, verify:

**Context Window (6 pts)**
- [ ] `returnOnlyIds` on lists
- [ ] `compact` mode on lists
- [ ] Cursor pagination
- [ ] `count_*` tools
- [ ] Field selection
- [ ] Summary-only batch responses

**Operations (5 pts)**
- [ ] `search_and_action` composite
- [ ] `get_or_create` patterns
- [ ] `dryRun` on destructive ops
- [ ] Internal batching
- [ ] Upsert support

**Safety (6 pts)**
- [ ] Batch size limits
- [ ] Two-stage deletion
- [ ] Confirmation tokens
- [ ] Rate limiting
- [ ] Dangerous operation markers
- [ ] Read-only mode support

**Target: 17+/17 points**

## Resources

- `references/patterns.md` - Detailed implementation patterns
- `references/platforms.md` - Platform-specific requirements (Email, Calendar, File Storage, CRM)
