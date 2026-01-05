# MCP Excellence Standard v1.0

> A comprehensive framework for building AI-agent-optimized Model Context Protocol (MCP) servers.

## Executive Summary

Most existing MCPs are built as simple API wrappers without considering the unique constraints of AI agents operating them. This standard defines best practices across 12 critical dimensions to create MCPs that are efficient, robust, and agent-friendly.

**Key Insight:** An MCP is not a human-facing API. It's an interface optimized for an AI agent with:
- Limited context window (tokens are expensive)
- No persistent memory between calls
- Pattern-matching decision making
- Tendency toward verbose exploration without guardrails

---

## Table of Contents

1. [Context Window Optimization](#1-context-window-optimization)
2. [Atomic vs Composite Operations](#2-atomic-vs-composite-operations)
3. [Error Handling & Resilience](#3-error-handling--resilience)
4. [Authentication & Multi-Account](#4-authentication--multi-account)
5. [Discovery & Introspection](#5-discovery--introspection)
6. [State & Caching](#6-state--caching)
7. [Resource Efficiency](#7-resource-efficiency)
8. [Semantic Grouping](#8-semantic-grouping)
9. [Audit & Observability](#9-audit--observability)
10. [Platform-Specific Patterns](#10-platform-specific-patterns)
11. [Agent-Friendly Output](#11-agent-friendly-output)
12. [Safety & Guardrails](#12-safety--guardrails)

---

## 1. Context Window Optimization

### Problem
LLMs have limited context windows. Returning 500 full email objects with bodies, headers, and metadata wastes thousands of tokens on data the agent may not need.

### Anti-Patterns ❌
```javascript
// Returns full objects regardless of need
search_emails({ query: "from:linkedin.com" })
// → Returns: [{id, subject, from, to, cc, bcc, date, body, headers, attachments...}, ...] × 50
// → ~5000 tokens consumed
```

### Best Practices ✅

#### 1.1 ID-Only Mode
Every list/search operation MUST support returning only IDs:

```javascript
search_emails({ 
  query: "from:linkedin.com",
  returnOnlyIds: true 
})
// → Returns: { count: 247, ids: ["abc123", "def456", ...] }
// → ~300 tokens consumed (94% reduction)
```

#### 1.2 Compact Mode
Support minimal representation with truncated fields:

```javascript
list_emails({ 
  folder: "inbox",
  limit: 100,
  compact: true 
})
// → Returns: { 
//     emails: [
//       { id: "abc", subject: "Meeting tomorrow at...", from: "john@acme.com", date: "2024-01-15" },
//       ...
//     ]
//   }
```

#### 1.3 Field Selection
Allow explicit field selection:

```javascript
get_emails({ 
  ids: ["abc", "def"],
  fields: ["id", "subject", "from", "date"]  // Only these fields returned
})
```

#### 1.4 Pagination with Cursors
Support cursor-based pagination for large datasets:

```javascript
list_emails({ 
  folder: "inbox",
  pageSize: 100,
  pageToken: null  // First page
})
// → Returns: { emails: [...], nextPageToken: "xyz789", hasMore: true }

list_emails({ 
  folder: "inbox",
  pageSize: 100,
  pageToken: "xyz789"  // Next page
})
```

#### 1.5 Count-Only Operations
Provide dedicated count tools for pre-flight decisions:

```javascript
count_emails({ query: "is:unread" })
// → Returns: { count: 1247 }
// Agent can now decide: "1247 is too many, let me narrow the query"
```

#### 1.6 Summary-Only Batch Responses
Batch operations MUST return summaries, not per-item results:

```javascript
// ❌ Bad: Returns list of all processed IDs
batch_delete({ ids: [...1000 ids...] })
// → Returns: { deleted: ["id1", "id2", ...1000 items...] }

// ✅ Good: Returns summary only
batch_delete({ ids: [...1000 ids...] })
// → Returns: { requested: 1000, deleted: 997, failed: 3 }
```

### Implementation Checklist
- [ ] All list operations support `returnOnlyIds` parameter
- [ ] All list operations support `compact` parameter
- [ ] All list operations support `fields` parameter or equivalent
- [ ] Pagination implemented with cursor tokens
- [ ] Dedicated `count_*` tools for each listable resource
- [ ] Batch operations return summaries, not item lists
- [ ] Default limits are conservative (10-25, not 100-500)

---

## 2. Atomic vs Composite Operations

### Problem
Agents make multiple round-trips when a single composite operation would suffice. Each round-trip adds latency and consumes context.

### Anti-Patterns ❌
```javascript
// Agent workflow: Archive all LinkedIn emails
const emails = await search_emails({ query: "from:linkedin.com" });  // Call 1
for (const email of emails) {
  await archive_email({ id: email.id });  // Calls 2-501
}
// Total: 501 API calls, massive context consumption
```

### Best Practices ✅

#### 2.1 Search-and-Action Composites
Combine search + action into single operations:

```javascript
search_and_action({
  query: "from:linkedin.com",
  action: "archive",
  maxResults: 500,
  dryRun: false
})
// → Returns: { query: "from:linkedin.com", action: "archive", processed: 247, failed: 0 }
// Total: 1 API call
```

#### 2.2 Get-or-Create Patterns
Idempotent creation that returns existing if present:

```javascript
get_or_create_label({ name: "Newsletters" })
// → Returns: { id: "Label_123", name: "Newsletters", created: false }
// (Returns existing label, doesn't fail or create duplicate)
```

#### 2.3 Upsert Operations
Update-or-insert semantics where applicable:

```javascript
upsert_contact({
  email: "john@acme.com",
  data: { name: "John Doe", phone: "+1234567890" }
})
// Creates if not exists, updates if exists
```

#### 2.4 Dry Run Mode
Preview operations before execution:

```javascript
search_and_action({
  query: "older_than:1y",
  action: "delete",
  dryRun: true  // Preview only
})
// → Returns: { dryRun: true, wouldProcess: 3847, query: "older_than:1y" }
// Agent can confirm with user before executing
```

#### 2.5 Bulk Operations with Internal Batching
Handle large operations internally:

```javascript
batch_move_emails({
  ids: [...5000 ids...],
  destination: "Archive",
  batchSize: 50  // Internal chunking
})
// MCP handles batching internally, returns single summary
// → Returns: { requested: 5000, moved: 4998, failed: 2 }
```

### Implementation Checklist
- [ ] `search_and_action` pattern for list + modify workflows
- [ ] `get_or_create` for all creatable resources
- [ ] `dryRun` parameter on all destructive/bulk operations
- [ ] Internal batching in bulk operations (not exposed to agent)
- [ ] Upsert semantics where API supports it

---

## 3. Error Handling & Resilience

### Problem
One failed item in a batch kills the entire operation. Errors are cryptic and unactionable.

### Anti-Patterns ❌
```javascript
// Entire batch fails if one item fails
batch_delete({ ids: ["valid1", "invalid2", "valid3"] })
// → Throws: Error: Message not found: invalid2
// valid1 and valid3 never processed

// Cryptic errors
modify_email({ id: "abc", labels: ["NonExistent"] })
// → Throws: Error: 400 Bad Request
```

### Best Practices ✅

#### 3.1 Partial Success Reporting
Continue processing after failures, report both:

```javascript
batch_delete({ ids: ["valid1", "invalid2", "valid3"] })
// → Returns: { 
//     success: 2, 
//     failed: 1, 
//     errors: [{ id: "invalid2", error: "Message not found" }]
//   }
```

#### 3.2 Structured Errors
Return machine-parseable error information:

```javascript
// ❌ Bad
{ error: "Something went wrong" }

// ✅ Good
{
  error: {
    code: "LABEL_NOT_FOUND",
    message: "Label 'NonExistent' does not exist",
    suggestion: "Use list_labels() to see available labels",
    recoverable: true
  }
}
```

#### 3.3 Internal Retry Logic
Handle transient failures automatically:

```python
# Internal to MCP - not exposed to agent
async def api_call_with_retry(fn, max_retries=3):
    for attempt in range(max_retries):
        try:
            return await fn()
        except RateLimitError:
            await asyncio.sleep(2 ** attempt)
        except TransientError:
            await asyncio.sleep(1)
    raise MaxRetriesExceeded()
```

#### 3.4 Graceful Degradation
Return partial data instead of failing completely:

```javascript
get_email_with_attachments({ id: "abc123" })
// If attachment fetch fails but email succeeds:
// → Returns: {
//     id: "abc123",
//     subject: "...",
//     body: "...",
//     attachments: null,
//     warnings: ["Could not fetch attachments: timeout"]
//   }
```

#### 3.5 Rate Limit Awareness
Expose rate limit status to agent:

```javascript
// Response headers or metadata
{
  data: { ... },
  rateLimit: {
    remaining: 45,
    resetAt: "2024-01-15T10:30:00Z",
    limit: 100
  }
}
```

### Implementation Checklist
- [ ] Batch operations continue after individual failures
- [ ] All errors include error code, message, and suggestion
- [ ] Internal retry for transient/rate-limit errors
- [ ] Partial results returned with warnings when possible
- [ ] Rate limit info exposed in responses

---

## 4. Authentication & Multi-Account

### Problem
Most MCPs assume single account. Token refresh breaks sessions. Credentials stored insecurely.

### Anti-Patterns ❌
```javascript
// Hardcoded single account
send_email({ to: "...", subject: "..." })
// Which account sends this? No way to specify.

// Token expired mid-session
search_emails({ query: "..." })
// → Error: Token expired. Please re-authenticate.
// Agent has no way to re-authenticate
```

### Best Practices ✅

#### 4.1 Multi-Account Support
All operations accept optional account identifier:

```javascript
list_accounts()
// → Returns: [
//     { id: "acc_1", email: "work@company.com", type: "work" },
//     { id: "acc_2", email: "personal@gmail.com", type: "personal" }
//   ]

send_email({ 
  accountId: "acc_1",  // Explicit account
  to: "...", 
  subject: "..." 
})
```

#### 4.2 Silent Token Refresh
Refresh tokens automatically without interrupting operations:

```python
async def ensure_valid_token(account_id: str):
    token = get_stored_token(account_id)
    if token.is_expired():
        new_token = await refresh_token(token.refresh_token)
        store_token(account_id, new_token)
        return new_token
    return token

# Called internally before every API request
```

#### 4.3 Clear Auth Status
Expose authentication state:

```javascript
get_auth_status()
// → Returns: {
//     accounts: [
//       { id: "acc_1", email: "...", status: "authenticated", expiresAt: "..." },
//       { id: "acc_2", email: "...", status: "requires_reauth", reason: "Token revoked" }
//     ]
//   }
```

#### 4.4 Device Flow Authentication
For headless/CLI environments:

```javascript
initiate_auth()
// → Returns: {
//     userCode: "ABCD-1234",
//     verificationUrl: "https://microsoft.com/devicelogin",
//     expiresIn: 900,
//     message: "Visit URL and enter code to authenticate"
//   }

complete_auth({ userCode: "ABCD-1234" })
// → Returns: { status: "success", accountId: "acc_new", email: "user@domain.com" }
```

#### 4.5 Secure Credential Storage
Use system keychain, not plaintext files:

```python
# ❌ Bad
with open("~/.myapp/credentials.json", "w") as f:
    json.dump(tokens, f)

# ✅ Good  
import keyring
keyring.set_password("myapp", account_id, json.dumps(tokens))
```

### Implementation Checklist
- [ ] All operations accept `accountId` parameter
- [ ] `list_accounts()` tool available
- [ ] Automatic token refresh (no agent intervention)
- [ ] `get_auth_status()` for checking auth state
- [ ] Device flow or equivalent for authentication
- [ ] Credentials stored in system keychain/secure storage

---

## 5. Discovery & Introspection

### Problem
Agent doesn't know what's possible without reading documentation. Enums and options are undiscoverable.

### Anti-Patterns ❌
```javascript
// What actions are available? Agent has to guess.
modify_email({ id: "abc", action: "archive" })  // Does "archive" work? Who knows.

// What labels exist? Must call separate API first.
move_email({ id: "abc", toLabel: "Work" })  // Is "Work" a valid label? 
```

### Best Practices ✅

#### 5.1 Capabilities Discovery
Single tool to understand what's available:

```javascript
get_capabilities()
// → Returns: {
//     resources: ["emails", "labels", "filters", "contacts"],
//     operations: {
//       emails: ["list", "get", "send", "delete", "modify", "search"],
//       labels: ["list", "create", "update", "delete"]
//     },
//     limits: {
//       maxBatchSize: 100,
//       maxResults: 500,
//       rateLimitPerMinute: 60
//     }
//   }
```

#### 5.2 Enum Discovery
Return valid options alongside operations:

```javascript
list_labels()
// → Returns: {
//     labels: [
//       { id: "INBOX", name: "Inbox", type: "system" },
//       { id: "Label_123", name: "Work", type: "user" }
//     ],
//     specialLabels: {
//       inbox: "INBOX",
//       sent: "SENT",
//       trash: "TRASH",
//       archive: "ARCHIVE"
//     }
//   }
```

#### 5.3 Rich Tool Descriptions
Include examples in tool descriptions:

```javascript
{
  name: "search_emails",
  description: `Search for emails using Gmail search syntax.
  
Examples:
- from:sender@example.com
- subject:"meeting notes"  
- has:attachment larger:5M
- after:2024/01/01 before:2024/02/01
- is:unread in:inbox

See: https://support.google.com/mail/answer/7190`,
  inputSchema: { ... }
}
```

#### 5.4 Schema Introspection
Understand data models programmatically:

```javascript
get_schema({ resource: "email" })
// → Returns: {
//     fields: {
//       id: { type: "string", description: "Unique message ID" },
//       subject: { type: "string", maxLength: 998 },
//       from: { type: "EmailAddress", description: "Sender" },
//       labels: { type: "array", items: "LabelId" }
//     },
//     actions: ["archive", "trash", "delete", "markRead", "markUnread"]
//   }
```

### Implementation Checklist
- [ ] `get_capabilities()` tool available
- [ ] List operations return enum/option values
- [ ] Tool descriptions include examples
- [ ] Schema introspection available
- [ ] Links to documentation in descriptions

---

## 6. State & Caching

### Problem
Agent re-fetches same data repeatedly. No incremental sync capability.

### Anti-Patterns ❌
```javascript
// Agent checks inbox every turn
list_emails({ folder: "inbox" })  // Turn 1
list_emails({ folder: "inbox" })  // Turn 2 (same data)
list_emails({ folder: "inbox" })  // Turn 3 (same data)
// 3x API calls for identical data
```

### Best Practices ✅

#### 6.1 ETag/Conditional Requests
Support conditional fetching:

```javascript
list_emails({ folder: "inbox" })
// → Returns: { emails: [...], etag: "abc123" }

list_emails({ folder: "inbox", ifNoneMatch: "abc123" })
// → Returns: { unchanged: true }  // No data transferred
// OR
// → Returns: { emails: [...], etag: "def456" }  // Changed
```

#### 6.2 Delta/Incremental Sync
Fetch only changes since last sync:

```javascript
sync_emails({ 
  folder: "inbox",
  since: "2024-01-15T10:30:00Z"  // Last sync time
})
// → Returns: {
//     added: [{ id: "new1", ... }],
//     modified: [{ id: "mod1", changes: {...} }],
//     deleted: ["del1", "del2"],
//     syncToken: "xyz789"
//   }
```

#### 6.3 Internal Caching with TTL
Cache frequently accessed data:

```python
# Internal implementation
@cached(ttl=300)  # 5 minute cache
async def get_labels(account_id: str):
    return await api.list_labels()

# Expose cache control to agent
invalidate_cache({ resource: "labels" })
```

#### 6.4 Watch/Subscribe (if platform supports)
Real-time updates instead of polling:

```javascript
watch_inbox({ 
  callback: "webhook_url",
  events: ["new_message", "message_deleted"]
})
// → Returns: { watchId: "watch_123", expiresAt: "..." }
```

### Implementation Checklist
- [ ] ETag support for conditional requests
- [ ] `since` / `after` parameters for incremental sync
- [ ] Internal caching for stable data (labels, folders)
- [ ] `invalidate_cache()` tool
- [ ] Delta responses where API supports it

---

## 7. Resource Efficiency

### Problem
Fetching full objects when summaries suffice. Loading attachments when only metadata needed.

### Anti-Patterns ❌
```javascript
// Fetches entire email including 10MB attachment
get_email({ id: "abc123" })
// → Returns: { ..., body: "...", attachments: [{ content: "BASE64_10MB..." }] }
```

### Best Practices ✅

#### 7.1 Lazy Loading / Expand Parameters
Only fetch nested resources when requested:

```javascript
get_email({ id: "abc123" })
// → Returns: { ..., body: "...", attachmentCount: 3 }  // No attachment content

get_email({ id: "abc123", expand: ["attachments"] })
// → Returns: { ..., attachments: [{ id: "att1", name: "doc.pdf", size: 10485760 }] }
// Still no content - just metadata

download_attachment({ emailId: "abc123", attachmentId: "att1" })
// → Actually downloads the file
```

#### 7.2 Exclude Heavy Fields by Default
Don't return body/content unless requested:

```javascript
list_emails({ folder: "inbox" })
// → Returns emails WITHOUT body (default)

list_emails({ folder: "inbox", includeBody: true })
// → Returns emails WITH body (explicit opt-in)
```

#### 7.3 Truncation with Indicators
Truncate long content with metadata:

```javascript
get_email({ id: "abc123", bodyMaxLength: 1000 })
// → Returns: { 
//     body: "First 1000 characters...",
//     bodyTruncated: true,
//     bodyTotalLength: 45000
//   }
```

#### 7.4 Streaming for Large Responses
Stream large data instead of buffering:

```python
async def export_emails(folder: str):
    async for batch in paginate_emails(folder):
        yield batch  # Stream batches instead of collecting all
```

### Implementation Checklist
- [ ] `expand` parameter for nested resources
- [ ] Heavy fields excluded by default
- [ ] `includeBody` / `includeContent` opt-in flags
- [ ] Truncation with metadata (`truncated: true`, `totalLength`)
- [ ] Attachment metadata vs content separated

---

## 8. Semantic Grouping

### Problem
50 tools with no organization. Agent picks wrong tool or doesn't discover the right one.

### Anti-Patterns ❌
```javascript
// Flat, inconsistent naming
tools: [
  "sendMail",
  "get_calendar_event", 
  "CreateContact",
  "deleteFile",
  "listEmails"
]
```

### Best Practices ✅

#### 8.1 Consistent Naming Convention
Use `{resource}_{action}` pattern:

```javascript
tools: [
  // Email operations
  "email_list",
  "email_get",
  "email_send",
  "email_delete",
  "email_search",
  
  // Calendar operations
  "calendar_list_events",
  "calendar_create_event",
  "calendar_delete_event",
  
  // Contact operations
  "contact_list",
  "contact_create",
  "contact_update"
]
```

#### 8.2 Composite Tools for Common Workflows
Reduce cognitive load with workflow-oriented tools:

```javascript
// Instead of: search → filter → batch_modify
"email_organize_by_sender"  // Composite

// Instead of: get_events → check_conflicts → create_event
"calendar_schedule_meeting"  // Composite with conflict checking
```

#### 8.3 Danger Indicators
Mark destructive operations clearly:

```javascript
{
  name: "email_delete_permanently",
  description: "⚠️ DESTRUCTIVE: Permanently deletes emails. Cannot be undone.",
  dangerous: true,
  requiresConfirmation: true
}
```

#### 8.4 Read/Write Separation
Group by mutation level:

```javascript
// Read operations (safe)
"email_list", "email_get", "email_search", "email_count"

// Write operations (mutations)
"email_send", "email_modify", "email_move"

// Delete operations (destructive)  
"email_trash", "email_delete"
```

### Implementation Checklist
- [ ] Consistent `{resource}_{action}` naming
- [ ] Composite tools for common workflows
- [ ] Clear danger/destructive indicators
- [ ] Logical grouping (read/write/delete)
- [ ] Short aliases for frequently used tools

---

## 9. Audit & Observability

### Problem
No visibility into what agent did. Can't undo mistakes. No action history.

### Best Practices ✅

#### 9.1 Action Logging
Track all mutations:

```javascript
get_action_history({ limit: 10 })
// → Returns: [
//     { timestamp: "...", action: "email_delete", params: {...}, result: "success" },
//     { timestamp: "...", action: "label_create", params: {...}, result: "success" }
//   ]
```

#### 9.2 Undo Capabilities
Where possible, support undo:

```javascript
email_trash({ id: "abc123" })
// → Returns: { id: "abc123", trashed: true, undoToken: "undo_xyz" }

email_untrash({ id: "abc123" })  
// OR
undo_action({ token: "undo_xyz" })
```

#### 9.3 Change Summaries
Include before/after in modification responses:

```javascript
email_modify({ id: "abc123", addLabels: ["Work"], removeLabels: ["Inbox"] })
// → Returns: {
//     id: "abc123",
//     changes: {
//       labelsAdded: ["Work"],
//       labelsRemoved: ["Inbox"],
//       labelsBefore: ["Inbox", "Unread"],
//       labelsAfter: ["Work", "Unread"]
//     }
//   }
```

#### 9.4 Dry Run / What-If Mode
Preview changes before applying:

```javascript
bulk_delete({ 
  query: "older_than:2y",
  dryRun: true 
})
// → Returns: {
//     dryRun: true,
//     wouldDelete: 3847,
//     oldestEmail: "2021-03-15",
//     sampleSubjects: ["Old newsletter...", "Receipt from..."]
//   }
```

### Implementation Checklist
- [ ] `get_action_history()` available
- [ ] Undo support for reversible operations
- [ ] Change summaries in modification responses
- [ ] `dryRun` mode on all bulk/destructive operations

---

## 10. Platform-Specific Patterns

### Email MCPs (Gmail, Outlook, etc.)

#### Required Tools
```
email_list              email_search_and_action
email_get               email_analyze_inbox
email_send              email_get_top_senders
email_delete            
email_search            label_list
email_count             label_create
email_collect_ids       label_get_stats
                        
batch_move_emails       filter_list
batch_delete_emails     filter_create
batch_modify_labels     filter_delete
```

#### Required Features
- Thread-aware operations (`threadId` support)
- Label/folder management
- Attachment handling (metadata vs download)
- Filter/rule creation
- Focused inbox / priority controls
- Search query syntax documentation

### Calendar MCPs (Google Calendar, Outlook, etc.)

#### Required Tools
```
event_list              event_respond (accept/decline/tentative)
event_get               
event_create            availability_check
event_update            free_busy_query
event_delete            
                        calendar_list
event_search            calendar_get_stats
```

#### Required Features
- Timezone handling (always explicit)
- Recurring event support (instances vs series)
- Attendee management
- Conflict detection before creation
- All-day event support
- Video conferencing integration

### File Storage MCPs (Google Drive, OneDrive, Dropbox)

#### Required Tools
```
file_list               file_share
file_get                file_unshare
file_upload             file_get_sharing
file_download           
file_delete             folder_create
file_move               folder_delete
file_copy               
                        quota_get
file_search             recent_activity
```

#### Required Features
- Chunked upload for large files
- Resumable uploads
- Version history access
- Sharing/permissions management
- Quota awareness
- MIME type handling
- Path vs ID addressing

### CRM/Database MCPs

#### Required Tools
```
record_list             record_bulk_create
record_get              record_bulk_update
record_create           record_bulk_delete
record_update           
record_delete           schema_get
                        schema_list_objects
record_search           
record_query            relationship_get
record_aggregate        relationship_create
```

#### Required Features
- Relationship traversal
- Complex query support
- Aggregation functions
- Schema introspection
- Bulk import/export
- Transaction support (where available)

---

## 11. Agent-Friendly Output

### Problem
Responses optimized for human reading, not machine parsing. Inconsistent schemas.

### Anti-Patterns ❌
```javascript
// Human-friendly but machine-hostile
return "Email sent successfully! The message ID is abc123. Have a great day!"

// Inconsistent schemas
list_emails → [{ id, subject, from }]
search_emails → [{ messageId, title, sender }]  // Different field names!
```

### Best Practices ✅

#### 11.1 JSON Responses Always
Never return formatted text for data:

```javascript
// ❌ Bad
return `Found 5 emails:
1. Subject: Meeting (from: john@acme.com)
2. Subject: Invoice (from: billing@vendor.com)
...`

// ✅ Good
return {
  count: 5,
  emails: [
    { id: "abc", subject: "Meeting", from: "john@acme.com" },
    { id: "def", subject: "Invoice", from: "billing@vendor.com" }
  ]
}
```

#### 11.2 Consistent Schemas
Same resource = same schema everywhere:

```javascript
// Email schema is IDENTICAL whether from list, search, or get
const EmailSchema = {
  id: string,
  subject: string,
  from: string,
  to: string[],
  date: string,  // ISO 8601 always
  labels: string[],
  isRead: boolean,
  hasAttachments: boolean
}
```

#### 11.3 Actionable IDs
Return IDs that can be used in subsequent calls:

```javascript
// ❌ Bad
{ message: "Created label 'Work'" }  // How do I reference it now?

// ✅ Good
{ id: "Label_123", name: "Work", created: true }  // ID ready for use
```

#### 11.4 Status Enums
Use enums, not free-text status:

```javascript
// ❌ Bad
{ status: "The email was sent successfully" }

// ✅ Good
{ status: "sent", id: "abc123" }
// status ∈ ["sent", "queued", "failed", "bounced"]
```

#### 11.5 Next-Step Hints
Guide agent to logical next actions:

```javascript
create_label({ name: "Newsletters" })
// → Returns: {
//     id: "Label_123",
//     name: "Newsletters",
//     nextActions: [
//       "Use email_move({ ids: [...], labelId: 'Label_123' }) to move emails here",
//       "Use filter_create({ action: { addLabels: ['Label_123'] }}) to auto-label"
//     ]
//   }
```

### Implementation Checklist
- [ ] All responses are JSON objects
- [ ] Consistent field names across all tools
- [ ] All created resources return usable IDs
- [ ] Status values are enums, not text
- [ ] Timestamps in ISO 8601 format
- [ ] Next-step hints in create/action responses

---

## 12. Safety & Guardrails

### Problem
Agent can accidentally delete everything. No limits on destructive operations.

### Anti-Patterns ❌
```javascript
// One bad query = disaster
delete_all_matching({ query: "*" })  // Deletes EVERYTHING

// No confirmation for bulk operations
batch_delete({ ids: [...10000 ids...] })  // Immediate execution
```

### Best Practices ✅

#### 12.1 Maximum Limits on Bulk Operations
Hard caps that can't be overridden:

```javascript
batch_delete({ ids: [...10000 ids...] })
// → Error: Maximum batch size is 1000. 
// → Use multiple calls or search_and_action with confirmation.
```

#### 12.2 Soft Delete Before Hard Delete
Two-stage deletion:

```javascript
email_trash({ id: "abc123" })     // Stage 1: Move to trash
email_delete({ id: "abc123" })    // Stage 2: Permanent delete (only works if trashed)
```

#### 12.3 Confirmation Tokens for Dangerous Operations
Require explicit confirmation:

```javascript
bulk_delete_old_emails({ olderThanDays: 365 })
// → Returns: {
//     confirmationRequired: true,
//     willDelete: 5847,
//     oldestDate: "2022-01-15",
//     confirmToken: "confirm_xyz789",
//     message: "This will permanently delete 5847 emails. Call again with confirmToken to proceed."
//   }

bulk_delete_old_emails({ olderThanDays: 365, confirmToken: "confirm_xyz789" })
// → Executes deletion
```

#### 12.4 Rate Limiting on Writes
Prevent runaway operations:

```javascript
// After 100 mutations in 1 minute:
// → Error: {
//     code: "RATE_LIMIT",
//     message: "Write rate limit exceeded. 100 operations per minute.",
//     retryAfter: 45
//   }
```

#### 12.5 Read-Only Mode Option
Support restricted operation:

```javascript
// In MCP configuration
{
  "readOnly": true  // All write operations disabled
}

// Attempting write:
email_send({ ... })
// → Error: MCP is in read-only mode. Write operations disabled.
```

#### 12.6 Scope Restrictions
Limit what can be accessed:

```javascript
// Configuration
{
  "scopes": {
    "labels": ["Inbox", "Sent"],  // Only these labels accessible
    "operations": ["read", "modify"],  // No delete
    "excludeLabels": ["Confidential"]  // Never touch these
  }
}
```

### Implementation Checklist
- [ ] Maximum batch sizes enforced
- [ ] Two-stage deletion (trash → delete)
- [ ] Confirmation tokens for bulk destructive ops
- [ ] Rate limiting on mutations
- [ ] Read-only mode support
- [ ] Scope/permission restrictions

---

## Appendix A: MCP Compliance Checklist

Use this checklist to evaluate MCP implementations:

### Context Window (Score: /6)
- [ ] `returnOnlyIds` mode on lists
- [ ] `compact` mode on lists
- [ ] Field selection support
- [ ] Cursor pagination
- [ ] `count_*` operations
- [ ] Summary-only batch responses

### Operations (Score: /5)
- [ ] `search_and_action` composites
- [ ] `get_or_create` patterns
- [ ] `dryRun` on destructive ops
- [ ] Internal batching
- [ ] Upsert support

### Errors (Score: /5)
- [ ] Partial success on batches
- [ ] Structured error responses
- [ ] Internal retry logic
- [ ] Graceful degradation
- [ ] Rate limit exposure

### Auth (Score: /5)
- [ ] Multi-account support
- [ ] Silent token refresh
- [ ] Auth status visibility
- [ ] Device flow support
- [ ] Secure credential storage

### Discovery (Score: /4)
- [ ] `get_capabilities()` tool
- [ ] Enum discovery in lists
- [ ] Rich tool descriptions
- [ ] Schema introspection

### Output (Score: /5)
- [ ] JSON responses always
- [ ] Consistent schemas
- [ ] Actionable IDs returned
- [ ] Status enums
- [ ] Next-step hints

### Safety (Score: /6)
- [ ] Batch size limits
- [ ] Two-stage deletion
- [ ] Confirmation tokens
- [ ] Rate limiting
- [ ] Read-only mode
- [ ] Scope restrictions

**Total Score: /36**

| Score | Rating |
|-------|--------|
| 30-36 | ⭐⭐⭐⭐⭐ Excellent - Production ready |
| 24-29 | ⭐⭐⭐⭐ Good - Minor improvements needed |
| 18-23 | ⭐⭐⭐ Adequate - Significant gaps |
| 12-17 | ⭐⭐ Poor - Major refactoring needed |
| 0-11  | ⭐ Failing - Rebuild recommended |

---

## Appendix B: Reference Implementations

### Email MCP Reference
See: `github.com/example/email-mcp-reference`

### Calendar MCP Reference  
See: `github.com/example/calendar-mcp-reference`

### Generic CRUD MCP Template
See: `github.com/example/mcp-template`

---

## Appendix C: Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-05 | Initial release |

---

## Contributing

This is a living document. Submit issues and PRs at:
`github.com/example/mcp-excellence-standard`

---

*"An MCP is not a human API. It's an interface for an AI agent with unique constraints and capabilities. Design accordingly."*
