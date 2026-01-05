# MCP Excellence - Detailed Implementation Patterns

This document contains expanded implementation patterns for building MCP Excellence Standard-compliant servers.

## Context Window Optimization Patterns

### ID-Only Mode Implementation

```typescript
interface ListParams {
  returnOnlyIds?: boolean;  // Returns { count, ids[] } only
  compact?: boolean;        // Minimal fields, truncated content
  fields?: string[];        // Explicit field selection
  pageSize?: number;        // Default: 25, Max: 100
  pageToken?: string;       // Cursor pagination
}

// Example implementation
async function listEmails(params: ListParams) {
  const emails = await api.fetchEmails(params);
  
  if (params.returnOnlyIds) {
    return {
      count: emails.total,
      ids: emails.items.map(e => e.id),
      nextPageToken: emails.nextToken
    };
  }
  
  if (params.compact) {
    return {
      count: emails.total,
      emails: emails.items.map(e => ({
        id: e.id,
        subject: e.subject.substring(0, 100),
        from: e.from,
        date: e.date,
        isRead: e.isRead
      })),
      nextPageToken: emails.nextToken
    };
  }
  
  // Full response
  return {
    count: emails.total,
    emails: emails.items,
    nextPageToken: emails.nextToken
  };
}
```

### Count Tool Implementation

```typescript
tools.add("count_emails", {
  description: "Returns count of emails matching query. Use before fetching to assess scope.",
  input: {
    query?: string,
    folder?: string,
    isUnread?: boolean
  },
  handler: async (params) => {
    const count = await api.countEmails(params);
    return { count };
  }
});
```

### Collect IDs Tool Implementation

```typescript
tools.add("collect_email_ids", {
  description: "Returns only IDs of emails matching criteria. Use with batch operations.",
  input: {
    query?: string,
    folder?: string,
    filterType?: "sender" | "domain" | "unread" | "older_than_days",
    filterValue?: string,
    maxResults?: number  // Default: 500
  },
  handler: async (params) => {
    const ids = await api.collectEmailIds(params);
    return { count: ids.length, ids };
  }
});
```

## Composite Operations Patterns

### Search and Action Implementation

```typescript
tools.add("search_and_action", {
  description: "Search for items and perform action in one operation. Supports dryRun for preview.",
  input: {
    query: string,
    action: "archive" | "trash" | "delete" | "markRead" | "markUnread" | "addLabel" | "removeLabel",
    actionParams?: {
      labelId?: string
    },
    maxResults?: number,
    dryRun?: boolean
  },
  handler: async (params) => {
    const matches = await api.search(params.query, params.maxResults);
    
    if (params.dryRun) {
      return {
        dryRun: true,
        query: params.query,
        action: params.action,
        wouldProcess: matches.length,
        sampleIds: matches.slice(0, 5).map(m => m.id)
      };
    }
    
    const results = await api.batchAction(matches.map(m => m.id), params.action, params.actionParams);
    
    return {
      query: params.query,
      action: params.action,
      processed: results.success,
      failed: results.failed
    };
  }
});
```

### Get or Create Implementation

```typescript
tools.add("get_or_create_label", {
  description: "Returns existing label if found, creates if not. Idempotent.",
  input: {
    name: string,
    color?: string
  },
  handler: async (params) => {
    // Try to find existing
    const existing = await api.findLabelByName(params.name);
    
    if (existing) {
      return {
        id: existing.id,
        name: existing.name,
        color: existing.color,
        created: false
      };
    }
    
    // Create new
    const created = await api.createLabel(params);
    return {
      id: created.id,
      name: created.name,
      color: created.color,
      created: true
    };
  }
});
```

## Error Handling Patterns

### Structured Error Response

```typescript
interface MCPError {
  code: string;           // Machine-parseable code
  message: string;        // Human-readable message
  suggestion?: string;    // How to fix
  recoverable: boolean;   // Can retry?
  retryAfter?: number;    // Seconds to wait if rate limited
}

function formatError(error: any): MCPError {
  if (error.status === 404) {
    return {
      code: "NOT_FOUND",
      message: `Resource not found: ${error.resourceId}`,
      suggestion: "Verify the ID exists using the list operation",
      recoverable: false
    };
  }
  
  if (error.status === 429) {
    return {
      code: "RATE_LIMIT",
      message: "Rate limit exceeded",
      suggestion: "Wait before retrying",
      recoverable: true,
      retryAfter: error.retryAfter || 60
    };
  }
  
  if (error.status === 400 && error.code === "INVALID_LABEL") {
    return {
      code: "LABEL_NOT_FOUND",
      message: `Label '${error.labelName}' does not exist`,
      suggestion: "Use list_labels() to see available labels",
      recoverable: false
    };
  }
  
  return {
    code: "UNKNOWN_ERROR",
    message: error.message || "An unknown error occurred",
    recoverable: false
  };
}
```

### Partial Success on Batches

```typescript
interface BatchResult {
  requested: number;
  success: number;
  failed: number;
  errors?: Array<{ id: string; error: string }>;
}

async function batchDelete(ids: string[]): Promise<BatchResult> {
  const results: BatchResult = {
    requested: ids.length,
    success: 0,
    failed: 0,
    errors: []
  };
  
  for (const id of ids) {
    try {
      await api.delete(id);
      results.success++;
    } catch (error) {
      results.failed++;
      results.errors.push({
        id,
        error: error.message
      });
    }
  }
  
  // Only include errors array if there were failures
  if (results.failed === 0) {
    delete results.errors;
  }
  
  return results;
}
```

### Internal Retry Logic

```typescript
async function apiCallWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      // Only retry transient errors
      if (error.status === 429 || error.status >= 500) {
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        await sleep(delay);
        continue;
      }
      
      // Non-retryable error
      throw error;
    }
  }
  
  throw lastError;
}
```

## Safety Patterns

### Batch Size Limits

```typescript
const MAX_BATCH_SIZE = 1000;
const DEFAULT_BATCH_SIZE = 50;

tools.add("batch_delete_emails", {
  input: {
    ids: string[],
    batchSize?: number
  },
  handler: async (params) => {
    if (params.ids.length > MAX_BATCH_SIZE) {
      throw {
        code: "BATCH_TOO_LARGE",
        message: `Maximum batch size is ${MAX_BATCH_SIZE}. Received ${params.ids.length}.`,
        suggestion: "Split into multiple calls or use search_and_action",
        recoverable: false
      };
    }
    
    const batchSize = params.batchSize || DEFAULT_BATCH_SIZE;
    // Process in internal chunks
    const results = await processInBatches(params.ids, batchSize, api.delete);
    
    return {
      requested: params.ids.length,
      deleted: results.success,
      failed: results.failed
    };
  }
});
```

### Confirmation Token Pattern

```typescript
tools.add("bulk_delete_old_emails", {
  description: "⚠️ DESTRUCTIVE: Permanently deletes emails older than specified days.",
  input: {
    olderThanDays: number,
    confirmToken?: string
  },
  handler: async (params) => {
    const query = `older_than:${params.olderThanDays}d`;
    const matches = await api.search(query);
    
    if (!params.confirmToken) {
      // Generate confirmation token
      const token = generateConfirmToken(query, matches.length);
      
      return {
        confirmationRequired: true,
        willDelete: matches.length,
        oldestDate: matches[matches.length - 1]?.date,
        sampleSubjects: matches.slice(0, 3).map(m => m.subject.substring(0, 50)),
        confirmToken: token,
        message: `This will permanently delete ${matches.length} emails. Call again with confirmToken to proceed.`
      };
    }
    
    // Validate token
    if (!validateConfirmToken(params.confirmToken, query, matches.length)) {
      throw {
        code: "INVALID_CONFIRM_TOKEN",
        message: "Confirmation token is invalid or expired",
        suggestion: "Request a new confirmation token",
        recoverable: true
      };
    }
    
    // Execute deletion
    const results = await batchDelete(matches.map(m => m.id));
    
    return {
      deleted: results.success,
      failed: results.failed
    };
  }
});
```

### Two-Stage Deletion

```typescript
tools.add("trash_email", {
  description: "Move email to trash. Can be undone with untrash_email.",
  input: { id: string },
  handler: async (params) => {
    await api.moveToTrash(params.id);
    return { id: params.id, trashed: true };
  }
});

tools.add("delete_email", {
  description: "⚠️ DESTRUCTIVE: Permanently delete email. Only works on trashed emails.",
  input: { id: string },
  handler: async (params) => {
    const email = await api.getEmail(params.id);
    
    if (!email.isTrashed) {
      throw {
        code: "NOT_TRASHED",
        message: "Email must be in trash before permanent deletion",
        suggestion: "Use trash_email first, then delete_email",
        recoverable: false
      };
    }
    
    await api.permanentDelete(params.id);
    return { id: params.id, deleted: true };
  }
});

tools.add("untrash_email", {
  description: "Restore email from trash.",
  input: { id: string },
  handler: async (params) => {
    await api.restoreFromTrash(params.id);
    return { id: params.id, restored: true };
  }
});
```

## Multi-Account Support Pattern

```typescript
tools.add("list_accounts", {
  description: "List all configured accounts",
  handler: async () => {
    const accounts = await auth.getAccounts();
    return {
      accounts: accounts.map(a => ({
        id: a.id,
        email: a.email,
        type: a.type,
        isDefault: a.isDefault
      }))
    };
  }
});

tools.add("get_auth_status", {
  description: "Check authentication status for all accounts",
  handler: async () => {
    const accounts = await auth.getAccounts();
    return {
      accounts: await Promise.all(accounts.map(async (a) => ({
        id: a.id,
        email: a.email,
        status: await auth.checkTokenStatus(a.id),
        expiresAt: a.tokenExpiry
      })))
    };
  }
});

// All operations accept optional accountId
tools.add("send_email", {
  input: {
    accountId?: string,  // Uses default if not specified
    to: string,
    subject: string,
    body: string
  },
  handler: async (params) => {
    const account = params.accountId 
      ? await auth.getAccount(params.accountId)
      : await auth.getDefaultAccount();
    
    await ensureValidToken(account.id);
    const result = await api.sendEmail(account, params);
    
    return { id: result.id, status: "sent" };
  }
});
```

## Discovery Pattern

```typescript
tools.add("get_capabilities", {
  description: "Returns available resources, operations, and limits",
  handler: async () => {
    return {
      resources: ["emails", "labels", "filters", "contacts"],
      operations: {
        emails: ["list", "get", "send", "delete", "modify", "search", "count"],
        labels: ["list", "create", "update", "delete"],
        filters: ["list", "create", "delete"]
      },
      limits: {
        maxBatchSize: 1000,
        maxResults: 500,
        rateLimitPerMinute: 60,
        maxAttachmentSize: "25MB"
      },
      features: {
        returnOnlyIds: true,
        compactMode: true,
        dryRun: true,
        multiAccount: true
      }
    };
  }
});
```
