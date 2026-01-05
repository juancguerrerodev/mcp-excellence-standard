# MCP Excellence - Platform-Specific Requirements

This document contains platform-specific patterns and requirements for different types of MCPs.

## Email MCPs (Gmail, Outlook, etc.)

### Required Tools

```
# Core Operations
email_list              # With returnOnlyIds, compact, pagination
email_get               # With expand parameter for attachments/headers
email_send              # With validation
email_delete            # Two-stage (trash first)
email_search            # With query syntax docs
email_count             # Count-only operation
email_collect_ids       # ID collection for batches

# Batch Operations
batch_move_emails       # With internal chunking
batch_delete_emails     # With confirmation tokens
batch_modify_labels     # Add/remove labels in bulk

# Composite Operations
email_search_and_action # Search + action with dryRun
email_analyze_inbox     # Top senders, domains, categories

# Label Management
label_list              # Include system vs user labels
label_create            # With get_or_create variant
label_get_stats         # Email counts per label

# Filter/Rule Management
filter_list
filter_create
filter_delete
```

### Required Features

- **Thread Support**: Include `threadId` in email responses, support thread-level operations
- **Search Syntax Documentation**: Include examples in tool description:
  ```
  from:sender@example.com
  subject:"meeting notes"
  has:attachment larger:5M
  after:2024/01/01 before:2024/02/01
  is:unread in:inbox
  ```
- **Label/Folder Management**: CRUD for labels, distinguish system vs user labels
- **Attachment Handling**: Separate metadata from content download
- **Focused Inbox**: If platform supports (Outlook), include focus/other management

### Email-Specific Optimizations

```typescript
// Inbox analytics - returns minimal data
tools.add("email_analyze_inbox", {
  description: "Analyze inbox patterns - top senders, domains, categories",
  input: {
    days?: number,  // Default: 30
    limit?: number  // Default: 10
  },
  handler: async (params) => {
    return {
      topSenders: [
        { email: "...", count: 50 },
        // ...
      ],
      topDomains: [
        { domain: "linkedin.com", count: 200 },
        // ...
      ],
      categories: {
        newsletters: 150,
        notifications: 80,
        personal: 30
      },
      unreadCount: 247,
      analyzedPeriod: { start: "...", end: "..." }
    };
  }
});
```

## Calendar MCPs (Google Calendar, Outlook, etc.)

### Required Tools

```
# Core Operations
event_list              # With date range, calendar filter
event_get               # With expand for attendees
event_create            # With conflict detection
event_update            # Partial updates supported
event_delete            # With notification options
event_search            # Full-text search

# Availability
availability_check      # Check conflicts before creating
free_busy_query         # Query free/busy for users

# Calendar Management
calendar_list           # List all calendars
calendar_get_stats      # Event counts, busy percentage

# RSVP
event_respond           # accept/decline/tentative
```

### Required Features

- **Timezone Handling**: Always explicit, never assume
  ```typescript
  {
    start: "2024-01-15T10:00:00",
    startTimezone: "America/New_York",  // REQUIRED
    end: "2024-01-15T11:00:00",
    endTimezone: "America/New_York"     // REQUIRED
  }
  ```
- **Recurring Events**: Handle instances vs series
  ```typescript
  // Get single instance
  event_get({ id: "event123", instanceDate: "2024-01-15" })
  
  // Update series vs instance
  event_update({ id: "event123", updateScope: "this" | "all" | "following" })
  ```
- **Attendee Management**: Add/remove attendees, RSVP status
- **Conflict Detection**: Check before creation
  ```typescript
  tools.add("event_create", {
    input: {
      // ...event details
      checkConflicts?: boolean  // Default: true
    },
    // Returns: { conflicts: [] } if checkConflicts and conflicts exist
  });
  ```
- **All-Day Events**: Proper handling without time components
- **Video Conferencing**: Auto-add Meet/Teams/Zoom links

### Calendar-Specific Patterns

```typescript
// Availability check before booking
tools.add("availability_check", {
  input: {
    attendees: string[],
    startTime: string,
    endTime: string,
    timezone: string
  },
  handler: async (params) => {
    const freeBusy = await api.queryFreeBusy(params);
    return {
      allAvailable: freeBusy.every(a => a.available),
      attendeeStatus: freeBusy.map(a => ({
        email: a.email,
        available: a.available,
        conflicts: a.conflicts
      })),
      suggestedAlternatives: await findAlternativeSlots(params)
    };
  }
});
```

## File Storage MCPs (Google Drive, OneDrive, Dropbox)

### Required Tools

```
# Core Operations
file_list               # With folder, returnOnlyIds, pagination
file_get                # Metadata only by default
file_upload             # Chunked for large files
file_download           # Separate from get
file_delete             # Two-stage (trash first)
file_move               # Between folders
file_copy               # With rename option
file_search             # Full-text and metadata search

# Sharing
file_share              # Create sharing link or add user
file_unshare            # Remove sharing
file_get_sharing        # Current sharing status

# Folder Operations
folder_create           # With path or parent ID
folder_delete           # Recursive option

# Storage
quota_get               # Used and available space
recent_activity         # Recent file changes
```

### Required Features

- **Chunked Upload**: For files > 5MB
  ```typescript
  tools.add("file_upload", {
    input: {
      parentId: string,
      name: string,
      content: string,      // Base64 for small files
      contentPath?: string, // Path for large files
      mimeType?: string
    }
    // Internally handles chunking for large files
  });
  ```
- **Resumable Uploads**: Store upload session for resume
- **Version History**: Access to file versions
- **Sharing/Permissions**: Full permission management
- **Quota Awareness**: Check before operations
- **MIME Type Handling**: Proper type detection and conversion
- **Path vs ID Addressing**: Support both
  ```typescript
  // Both should work:
  file_get({ id: "abc123" })
  file_get({ path: "/Documents/report.pdf" })
  ```

### File-Specific Patterns

```typescript
// Quota-aware upload
tools.add("file_upload", {
  handler: async (params) => {
    const quota = await api.getQuota();
    const fileSize = calculateSize(params.content);
    
    if (fileSize > quota.remaining) {
      throw {
        code: "QUOTA_EXCEEDED",
        message: `File size (${formatSize(fileSize)}) exceeds available quota (${formatSize(quota.remaining)})`,
        suggestion: "Delete files or upgrade storage",
        recoverable: false
      };
    }
    
    // Proceed with upload
  }
});
```

## CRM/Database MCPs

### Required Tools

```
# Core CRUD
record_list             # With filters, pagination, fields selection
record_get              # With relationship expansion
record_create           # With validation
record_update           # Partial updates
record_delete           # Soft delete preferred
record_search           # Full-text search
record_query            # Complex query support
record_aggregate        # Count, sum, avg, etc.

# Bulk Operations
record_bulk_create      # With validation, partial success
record_bulk_update      # With confirmation for large batches
record_bulk_delete      # With confirmation tokens

# Schema
schema_get              # Single object schema
schema_list_objects     # All available objects

# Relationships
relationship_get        # Related records
relationship_create     # Link records
```

### Required Features

- **Relationship Traversal**: Navigate object relationships
  ```typescript
  record_get({
    object: "Contact",
    id: "123",
    expand: ["Account", "Opportunities", "Tasks"]
  })
  ```
- **Complex Query Support**: Filter, sort, aggregate
  ```typescript
  record_query({
    object: "Opportunity",
    filters: [
      { field: "Amount", operator: ">=", value: 10000 },
      { field: "Stage", operator: "in", value: ["Negotiation", "Closed Won"] }
    ],
    orderBy: [{ field: "CloseDate", direction: "desc" }],
    limit: 100
  })
  ```
- **Aggregation Functions**: Count, sum, avg, min, max
- **Schema Introspection**: Discover fields, types, relationships
- **Bulk Import/Export**: Handle large datasets
- **Transaction Support**: Where available, atomic operations

### CRM-Specific Patterns

```typescript
// Schema introspection
tools.add("schema_get", {
  input: { object: string },
  handler: async (params) => {
    const schema = await api.describeObject(params.object);
    return {
      name: schema.name,
      label: schema.label,
      fields: schema.fields.map(f => ({
        name: f.name,
        type: f.type,
        required: f.required,
        picklistValues: f.picklistValues
      })),
      relationships: schema.relationships.map(r => ({
        name: r.name,
        relatedObject: r.relatedObject,
        type: r.type  // "lookup" | "masterDetail" | "many2many"
      }))
    };
  }
});

// Aggregation
tools.add("record_aggregate", {
  input: {
    object: string,
    groupBy?: string[],
    aggregations: Array<{
      field: string,
      function: "count" | "sum" | "avg" | "min" | "max"
    }>,
    filters?: Filter[]
  },
  handler: async (params) => {
    return await api.aggregate(params);
  }
});
```

## Common Patterns Across All Platforms

### Analytics Tool Pattern

Every MCP should have an analytics/stats tool:

```typescript
tools.add("get_{resource}_stats", {
  description: "Get summary statistics without fetching full data",
  handler: async () => {
    return {
      totalCount: 1500,
      byCategory: { ... },
      byStatus: { ... },
      recentActivity: { ... }
    };
  }
});
```

### Discovery Tool Pattern

```typescript
tools.add("get_capabilities", {
  description: "Discover available operations and limits",
  handler: async () => {
    return {
      resources: [...],
      operations: { ... },
      limits: { ... },
      features: { ... }
    };
  }
});
```

### Health Check Pattern

```typescript
tools.add("health_check", {
  description: "Check API connectivity and authentication",
  handler: async () => {
    const start = Date.now();
    await api.ping();
    return {
      status: "healthy",
      latencyMs: Date.now() - start,
      authStatus: await auth.checkStatus()
    };
  }
});
```
