# 🏆 MCP Excellence Standard

> **The comprehensive framework for building AI-agent-optimized Model Context Protocol (MCP) servers.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![MCP Version](https://img.shields.io/badge/MCP-1.0-blue.svg)](https://modelcontextprotocol.io)
[![Standard Version](https://img.shields.io/badge/Standard-v1.0-green.svg)](./MCP_EXCELLENCE_STANDARD.md)

---

## 🎯 The Problem

Most existing MCPs are built as **simple API wrappers** without considering the unique constraints of AI agents:

- **Limited context window** → Tokens are expensive
- **No persistent memory** → State must be managed differently  
- **Pattern-matching decisions** → Needs clear, consistent interfaces
- **Tendency toward verbose exploration** → Requires guardrails

The result? MCPs that waste tokens, require multiple round-trips, and lack safety controls.

## 💡 The Solution

This standard defines **best practices across 12 critical dimensions** to create MCPs that are efficient, robust, and agent-friendly.

---

## 📖 What's Included

| File | Description |
|------|-------------|
| [MCP_EXCELLENCE_STANDARD.md](./MCP_EXCELLENCE_STANDARD.md) | The complete standard with detailed patterns and anti-patterns |
| [skills/mcp-excellence/](./skills/mcp-excellence/) | Claude skill for building standard-compliant MCPs (Anthropic format) |
| [checklists/compliance-checklist.md](./checklists/compliance-checklist.md) | Quick evaluation checklist for any MCP |

---

## 🔑 The 12 Pillars

### 1. Context Window Optimization
- `returnOnlyIds` mode for lists
- `compact` mode with minimal fields
- Cursor-based pagination
- Count-only operations
- Summary-only batch responses

### 2. Atomic vs Composite Operations
- `search_and_action` composites
- `get_or_create` patterns
- `dryRun` preview mode
- Internal batching

### 3. Error Handling & Resilience
- Partial success reporting
- Structured error responses
- Internal retry logic
- Graceful degradation

### 4. Authentication & Multi-Account
- Account ID parameters
- Silent token refresh
- Device flow support
- Secure credential storage

### 5. Discovery & Introspection
- `get_capabilities()` tool
- Enum discovery
- Rich tool descriptions
- Schema introspection

### 6. State & Caching
- ETag/conditional requests
- Delta/incremental sync
- Internal caching with TTL

### 7. Resource Efficiency
- Lazy loading / expand parameters
- Exclude heavy fields by default
- Truncation with metadata

### 8. Semantic Grouping
- `{resource}_{action}` naming
- Danger indicators
- Read/write separation

### 9. Audit & Observability
- Action history logging
- Undo capabilities
- Change summaries

### 10. Platform-Specific Patterns
- Email MCP requirements
- Calendar MCP requirements
- File Storage MCP requirements
- CRM/Database MCP requirements

### 11. Agent-Friendly Output
- JSON responses always
- Consistent schemas
- Actionable IDs
- Status enums

### 12. Safety & Guardrails
- Maximum batch sizes
- Two-stage deletion
- Confirmation tokens
- Rate limiting
- Read-only mode

---

## 📊 Compliance Scoring

Use the [compliance checklist](./checklists/compliance-checklist.md) to evaluate any MCP:

| Score | Rating |
|-------|--------|
| 30-36 | ⭐⭐⭐⭐⭐ Excellent - Production ready |
| 24-29 | ⭐⭐⭐⭐ Good - Minor improvements needed |
| 18-23 | ⭐⭐⭐ Adequate - Significant gaps |
| 12-17 | ⭐⭐ Poor - Major refactoring needed |
| 0-11  | ⭐ Failing - Rebuild recommended |

---

## 🛠️ Using the Claude Skill

The included [Claude skill](./skills/mcp-excellence/) follows **Anthropic's official skill format** with proper YAML frontmatter and progressive disclosure. 

### Skill Structure
```
skills/mcp-excellence/
├── SKILL.md              # Main skill file with YAML frontmatter
└── references/
    ├── patterns.md       # Detailed implementation patterns
    └── platforms.md      # Platform-specific requirements
```

### Installation

**Claude Code:** Copy `skills/mcp-excellence/` to `~/.claude/skills/`

**Claude.ai:** Zip the folder and upload via Settings > Features

### Key Features:
- Automatic context window optimization
- Built-in batch operations
- Error handling patterns
- Safety guardrails by default

---

## 🚀 Quick Start

### Evaluating an Existing MCP

1. Open the [Compliance Checklist](./checklists/compliance-checklist.md)
2. Score each category
3. Identify gaps and prioritize improvements

### Building a New MCP

1. Install the skill (see Installation above)
2. Ask Claude: *"Create an MCP for [API name] following the MCP Excellence Standard"*
3. Provide your API documentation
4. Claude will generate a standard-compliant MCP

### Upgrading an Existing MCP

1. Install the skill
2. Ask Claude: *"Review this MCP and improve it following the MCP Excellence Standard"*
3. Share your existing MCP code
4. Claude will identify gaps and implement improvements

---

## 💡 Pro Tips

> **The skill works for both creating NEW MCPs and IMPROVING existing ones!**

### Example Prompts

**Create a new MCP:**
```
"Create a Gmail MCP following the MCP Excellence Standard with 
context window optimization and batch operations"
```

**Improve an existing MCP:**
```
"Review this MCP code and enhance it following the MCP Excellence 
Standard. Add returnOnlyIds, compact mode, and search_and_action."
```

**Audit an MCP:**
```
"Score this MCP against the MCP Excellence Standard compliance 
checklist and tell me what's missing."
```

**Add specific features:**
```
"Add context window optimization to this MCP - I need returnOnlyIds,
compact mode, and count operations."
```

### When to Use Each Resource

| Resource | Use For |
|----------|--------|
| **Skill** | Let Claude auto-apply best practices while coding |
| **Standard** | Deep reference for understanding the "why" |
| **Checklist** | Quick audit of any MCP (yours or third-party) |

---

## 📚 Examples

### Before (Anti-Pattern)
```javascript
// Returns 500 full email objects
const emails = await mcp.search_emails({ query: "from:newsletter" });
// Then loops to delete each one
for (const email of emails) {
  await mcp.delete_email({ id: email.id });
}
// Result: 501 API calls, ~50,000 tokens consumed
```

### After (Excellence Standard)
```javascript
// Single composite operation with preview
const result = await mcp.search_and_action({
  query: "from:newsletter",
  action: "trash",
  dryRun: false
});
// Result: 1 API call, ~100 tokens consumed
// Returns: { processed: 247, failed: 0 }
```

---

## 🤝 Contributing

This is a living standard. Contributions welcome!

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

### Areas for Contribution:
- New platform-specific patterns
- Reference implementations
- Linting tools
- Additional language examples

---

## 📄 License

MIT License - See [LICENSE](./LICENSE) for details.

---

## 👤 Author

**Juan C. Guerrero**
- 🌐 [juancguerrero.com](https://juancguerrero.com)
- 🐦 [@JuanCGuerreroE](https://twitter.com/JuanCGuerreroE)
- 💼 [GitHub](https://github.com/juancguerrerodev)

Founder of [Blockchain Jungle](https://blockchainJungle.xyz) | Co-founder of [Dojo Coding](https://dojocoding.io) & [All Time High](https://alltimehigh.academy)

---

## 🙏 Acknowledgments

This standard was developed through hands-on experience building and enhancing multiple MCPs, identifying patterns that waste AI agent resources, and synthesizing best practices that the industry was lacking.

---

*"An MCP is not a human API. It's an interface for an AI agent with unique constraints and capabilities. Design accordingly."*
