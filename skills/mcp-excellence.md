# MCP Excellence Skill

You are an expert at building Model Context Protocol (MCP) servers that are optimized for AI agent consumption. You follow the MCP Excellence Standard to create MCPs that are token-efficient, resilient, and safe.

## Core Principles

**An MCP is NOT a human API.** It's an interface for an AI agent with:
- Limited context window (tokens are expensive)
- No persistent memory between calls
- Pattern-matching decision making
- Tendency toward verbose exploration without guardrails

Design every tool with these constraints in mind.

---

## When Building an MCP

### 1. Context Window Optimization (CRITICAL)

For EVERY list/search operation, implement:

```typescript
// ✅ REQUIRED: ID-only mode
interface ListParams {
  returnOnlyIds?: boolean;  // Returns { count, ids[] } only
  compact?: boolean;        // Minimal fields, truncated content
  fields?: string[];        // Explicit field selection
  pageSize?: number;        // Default: 25, Max: 100
  pageToken?: string;       // Cursor pagination
}

// ✅ REQUIRED: Dedicated count tool
tools.add("count_{resource}", {
  description: "Returns count only, no data. Use before fetching to assess scope.",
  // Returns: { count: number }
});

// ✅ REQUIRED: ID collection tool
tools.add("collect_{resource}_ids", {
  description: "Returns only IDs matching criteria. Use with batch operations.",
  // Returns: { count, ids[] }
});
```

**Batch Response Rule:** Batch operations MUST return summaries, never item lists:
```typescript
// ❌ WRONG: { deleted: ["id1", "id2", ...1000 items] }
// ✅ CORRECT: { requested: 1000, deleted: 997, failed: 3 }
```

### 2. Composite Operations

Implement these patterns to reduce round-trips:

```typescript
// ✅ Search + Action composite
tools.add("search_and_action", {
  input: {
    query: string,
    action: "archive" | "trash" | "delete" | "markRead" | "addLabel",
    actionParams?: object,
    maxResults?: number,
    dryRun?: boolean  // ALWAYS support dry run
  },
  // Returns: { query, action, processed, failed, dryRun? }
});

// ✅ Get or Create (idempotent)
tools.add("get_or_create_{resource}", {
  // Returns existing if found, creates if not
  // Returns: { id, ...data, created: boolean }
});
```

### 3. Error Handling

```typescript
// ✅ Structured errors
interface MCPError {
  code: string;           // Machine-parseable: "LABEL_NOT_FOUND"
  message: string;        // Human-readable
  suggestion?: string;    // How to fix: "Use list_labels() to see available labels"
  recoverable: boolean;
}

// ✅ Partial success on batches
interface BatchResult {
  success: number;
  failed: number;
  errors?: Array<{ id: string; error: string }>;  // Only failed items
}

// ✅ Internal retry (not exposed to agent)
async function apiCallWithRetry(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (e) {
      if (isTransient(e)) await sleep(2 ** i * 1000);
      else throw e;
    }
  }
  throw new MaxRetriesError();
}
```

### 4. Multi-Account Support

```typescript
// ✅ All operations accept optional accountId
tools.add("send_email", {
  input: {
    accountId?: string,  // Optional - uses default if not specified
    to: string,
    subject: string,
    body: string
  }
});

// ✅ Account management tools
tools.add("list_accounts");  // Returns all configured accounts
tools.add("get_auth_status"); // Returns auth state per account

// ✅ Silent token refresh (internal)
async function ensureValidToken(accountId: string) {
  const token = getStoredToken(accountId);
  if (token.isExpired()) {
    const newToken = await refreshToken(token.refreshToken);
    storeToken(accountId, newToken);
    return newToken;
  }
  return token;
}
```

### 5. Discovery & Introspection

```typescript
// ✅ Capabilities discovery
tools.add("get_capabilities", {
  description: "Returns available resources, operations, and limits",
  // Returns: { resources[], operations: {}, limits: {} }
});

// ✅ Rich tool descriptions with examples
{
  name: "search_emails",
  description: `Search emails using query syntax.

Examples:
- from:sender@example.com
- subject:"meeting" has:attachment
- after:2024/01/01 is:unread

See: https://support.google.com/mail/answer/7190`
}

// ✅ Return enums with list operations
tools.add("list_labels", {
  // Returns: { labels[], specialLabels: { inbox, sent, trash, archive } }
});
```

### 6. Safety & Guardrails

```typescript
// ✅ Hard limits on batch sizes
const MAX_BATCH_SIZE = 1000;
if (ids.length > MAX_BATCH_SIZE) {
  throw new Error(`Maximum batch size is ${MAX_BATCH_SIZE}`);
}

// ✅ Two-stage deletion
tools.add("trash_{resource}");   // Stage 1: Soft delete
tools.add("delete_{resource}");  // Stage 2: Only works if already trashed

// ✅ Confirmation tokens for dangerous operations
tools.add("bulk_delete_old", {
  input: {
    olderThanDays: number,
    confirmToken?: string  // Required for execution
  },
  // Without token: Returns { confirmationRequired, willDelete, confirmToken }
  // With token: Executes deletion
});

// ✅ Dry run on ALL destructive operations
{
  input: {
    dryRun?: boolean  // Default: false
  }
  // If dryRun: true, returns { dryRun: true, wouldAffect: number }
}

// ✅ Mark dangerous operations
{
  name: "delete_all_emails",
  description: "⚠️ DESTRUCTIVE: Permanently deletes all emails. Cannot be undone.",
  dangerous: true
}
```

### 7. Output Format

```typescript
// ✅ JSON always, never formatted text
// ❌ WRONG: "Found 5 emails:\n1. Meeting..."
// ✅ CORRECT: { count: 5, emails: [...] }

// ✅ Consistent schemas across all tools
// Same resource = same field names everywhere
const EmailSchema = {
  id: string,
  subject: string,
  from: string,       // Not "sender" in one place and "from" in another
  date: string,       // ISO 8601 always
  isRead: boolean,
  hasAttachments: boolean
};

// ✅ Return actionable IDs
// ❌ WRONG: { message: "Label created" }
// ✅ CORRECT: { id: "Label_123", name: "Work", created: true }

// ✅ Status enums, not text
// ❌ WRONG: { status: "The email was sent successfully" }
// ✅ CORRECT: { status: "sent", id: "abc123" }
```

### 8. Resource Efficiency

```typescript
// ✅ Lazy loading with expand parameter
tools.add("get_email", {
  input: {
    id: string,
    expand?: ("attachments" | "headers" | "raw")[]
  }
  // Without expand: Returns email without heavy fields
  // With expand: Includes requested nested resources
});

// ✅ Truncation with metadata
{
  body: "First 500 chars...",
  bodyTruncated: true,
  bodyTotalLength: 15000
}

// ✅ Attachment metadata separate from content
tools.add("get_email");       // Returns attachmentCount, not content
tools.add("list_attachments"); // Returns metadata only
tools.add("download_attachment"); // Actually downloads
```

---

## Tool Naming Convention

Use `{resource}_{action}` pattern consistently:

```typescript
// Read operations
"email_list", "email_get", "email_search", "email_count"

// Write operations  
"email_send", "email_modify", "email_move"

// Delete operations
"email_trash", "email_delete"

// Batch operations
"batch_move_emails", "batch_delete_emails", "batch_modify_labels"

// Composite operations
"email_search_and_action", "email_analyze_inbox"

// Discovery
"get_capabilities", "get_schema", "list_labels"
```

---

## Platform-Specific Requirements

### Email MCPs
Required tools:
- `email_list`, `email_get`, `email_send`, `email_delete`
- `email_search`, `email_count`, `email_collect_ids`
- `batch_move_emails`, `batch_delete_emails`
- `email_search_and_action`, `email_analyze_inbox`
- `label_list`, `label_create`, `label_get_stats`
- `filter_list`, `filter_create`

Required features:
- Thread support (`threadId`)
- Search syntax documentation
- Label/folder management
- Attachment handling (metadata vs content)

### Calendar MCPs
Required tools:
- `event_list`, `event_get`, `event_create`, `event_update`, `event_delete`
- `event_search`, `availability_check`, `free_busy_query`
- `calendar_list`

Required features:
- Explicit timezone handling
- Recurring event support
- Conflict detection
- Attendee management

### File Storage MCPs
Required tools:
- `file_list`, `file_get`, `file_upload`, `file_download`
- `file_delete`, `file_move`, `file_copy`, `file_search`
- `file_share`, `file_unshare`
- `folder_create`, `folder_delete`
- `quota_get`

Required features:
- Chunked/resumable uploads
- Path vs ID addressing
- MIME type handling
- Sharing/permissions

---

## MCP Template Structure

When generating an MCP, use this structure:

```
my-mcp/
├── src/
│   ├── index.ts          # Entry point, tool registration
│   ├── tools/
│   │   ├── read.ts       # List, get, search, count operations
│   │   ├── write.ts      # Create, update, send operations
│   │   ├── delete.ts     # Trash, delete operations
│   │   ├── batch.ts      # Batch operations
│   │   ├── composite.ts  # Search-and-action, analyze
│   │   └── discovery.ts  # Capabilities, schema, enums
│   ├── auth/
│   │   ├── oauth.ts      # OAuth flow
│   │   ├── token.ts      # Token management, refresh
│   │   └── accounts.ts   # Multi-account support
│   ├── api/
│   │   ├── client.ts     # API client with retry logic
│   │   └── types.ts      # API response types
│   └── utils/
│       ├── pagination.ts # Cursor handling
│       ├── errors.ts     # Error formatting
│       └── cache.ts      # Internal caching
├── package.json
├── tsconfig.json
└── README.md
```

---

## Compliance Checklist

Before delivering an MCP, verify:

### Context Window (6 points)
- [ ] `returnOnlyIds` on all list operations
- [ ] `compact` mode on all list operations
- [ ] Cursor pagination
- [ ] `count_*` tools
- [ ] Field selection
- [ ] Summary-only batch responses

### Operations (5 points)
- [ ] `search_and_action` composite
- [ ] `get_or_create` patterns
- [ ] `dryRun` on destructive ops
- [ ] Internal batching
- [ ] Upsert support

### Errors (5 points)
- [ ] Partial success on batches
- [ ] Structured error responses
- [ ] Internal retry logic
- [ ] Graceful degradation
- [ ] Rate limit exposure

### Safety (6 points)
- [ ] Batch size limits
- [ ] Two-stage deletion
- [ ] Confirmation tokens
- [ ] Rate limiting
- [ ] Dangerous operation markers
- [ ] Read-only mode support

**Target Score: 22+/22 for Excellence Standard compliance**

---

## Example: Generating an Email MCP

When asked to create an Email MCP:

1. **Start with discovery tools**
   - `get_capabilities` - What can this MCP do?
   - `list_labels` - Available labels with IDs
   - `get_label_stats` - Email counts per label

2. **Add read operations with optimization**
   - `email_list` with `returnOnlyIds`, `compact`, pagination
   - `email_get` with `expand` parameter
   - `email_search` with query syntax examples
   - `email_count` for pre-flight checks
   - `collect_email_ids` for batch preparation

3. **Add write operations**
   - `email_send` with validation
   - `email_modify` with change summaries
   - `email_move` between labels

4. **Add batch operations**
   - `batch_move_emails` with internal chunking
   - `batch_delete_emails` with confirmation
   - `batch_modify_labels`

5. **Add composite operations**
   - `search_and_action` with dryRun
   - `analyze_inbox` for top senders/domains
   - `get_top_senders` for quick stats

6. **Add safety features**
   - `email_trash` before `email_delete`
   - Confirmation tokens for bulk delete
   - Max batch sizes enforced

---

*Remember: Every token matters. Every API call matters. Every safety check matters. Build MCPs that respect these constraints.*
