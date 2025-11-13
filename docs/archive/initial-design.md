# Claude Sonnet 4.5 Complete - Design Document

**Date:** 2025-11-12
**Author:** Enhanced by AI
**Version:** 3.0.0
**Status:** Design Complete - Ready for Implementation

---

## Executive Summary

This document describes the design for a production-grade OpenWebUI pipe function that implements **Claude Sonnet 4.5** with comprehensive support for all modern Anthropic API features available as of November 2025. The implementation focuses on:

- **Rich UX**: Progressive disclosure of thinking, verbose web search display, embedded citation cards
- **Modern Features**: Extended thinking, prompt caching, web search, skills, code execution
- **Correct Implementation**: Fixes critical bugs in citation extraction and event handling
- **User Control**: Extensive valve configuration for admins and users

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Feature Set](#feature-set)
3. [Valve Configuration](#valve-configuration)
4. [Streaming Implementation](#streaming-implementation)
5. [API Request Construction](#api-request-construction)
6. [Error Handling](#error-handling)
7. [Bug Fixes from v2.0.3](#bug-fixes-from-v203)
8. [Implementation Checklist](#implementation-checklist)

---

## Architecture Overview

### Core Design Principles

1. **Single Model ID**: Use `claude-sonnet-4-5-20250929` for all requests
2. **Streaming-First**: All responses stream by default with progressive disclosure
3. **Event-Driven UX**: Leverage OpenWebUI's event emitter for rich status updates
4. **Dynamic Beta Headers**: Compose beta headers based on enabled features
5. **Feature Toggles**: Extensive valve system for granular control

### Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| **No MCP Integration** | MCP is desktop-app specific; users can use Claude Desktop separately |
| **No Persistent Memory** | Memory tool requires complex client-side handlers; users can attach "memory files" as documents |
| **Skills via Code Execution** | Skills require code execution beta + skill IDs |
| **Inline Citation Cards** | Richer UX than OpenWebUI's native citation chips; requires `self.citation = False` |
| **Always Use Interleaved Thinking** | When extended thinking + tools enabled, automatically enable interleaved thinking for best quality |

### Component Structure

```
Pipe Class
├── Valves (Admin Configuration)
├── UserValves (Per-User Settings)
├── __init__() - Initialize with self.citation = False
├── pipes() - Return model list
├── pipe() - Main entry point
├── _get_headers() - Dynamic beta header composition
├── _configure_tools() - Build tools array
├── _configure_thinking() - Thinking configuration
├── _apply_caching() - Cache breakpoint placement
├── _calculate_max_tokens() - Account for thinking budget
├── stream_response() - Streaming with progressive disclosure
└── non_stream_response() - Non-streaming fallback
```

---

## Feature Set

### ✅ Implemented Features

#### 1. Extended Thinking
- **API Support**: Claude Sonnet 4.5, 4, Haiku 4.5, Opus 4.1, Opus 4
- **Budget Range**: 1,024 - 16,000 tokens (configurable, can go higher with batch API)
- **Interleaved Thinking**: Auto-enabled when thinking + tools are both active
- **Beta Header**: `interleaved-thinking-2025-05-14`
- **Constraint**: `budget_tokens < max_tokens` (except with interleaved thinking)

#### 2. Prompt Caching
- **Cache Types**:
  - Standard (5 min TTL) - `prompt-caching-2024-07-31`
  - Extended (1 hour TTL) - `extended-cache-ttl-2025-04-11`
- **Cache Breakpoints**:
  - System prompt (last item in system array)
  - 2nd-to-last user message (for stable cache points)
- **Cost Savings**: Up to 90% on cached tokens (10% of base price)

#### 3. Web Search
- **API Version**: `web_search_20250305`
- **Beta Header**: `web-search-2025-03-05`
- **Max Uses**: 1-20 configurable searches per request
- **Domain Control**: Allow lists OR block lists (not both)
- **Pricing**: $10 per 1,000 searches + token costs
- **Citations**: Included in text block `citations` array

#### 4. Skills System
- **Pre-built Skills**:
  - `xlsx` - Excel spreadsheet read/write with formulas
  - `pptx` - PowerPoint presentation generation
  - `docx` - Word document creation/editing
  - `pdf` - PDF generation with forms
- **Custom Skills**: User-provided skill IDs via valve
- **Requirements**:
  - `code-execution-2025-08-25`
  - `skills-2025-10-02`
  - `files-api-2025-04-14`

#### 5. Code Execution
- **Beta Header**: `code-execution-2025-08-25`
- **Language**: Python only
- **Container**: Skills run in isolated execution environment
- **Limitations**: No network access, no runtime package installation

#### 6. PDF Support
- **Methods**: Base64, URL, or File ID
- **Max Size**: 32MB
- **Max Pages**: 100 pages
- **Format**: Passed as document content blocks

#### 7. Files API
- **Beta Header**: `files-api-2025-04-14`
- **Purpose**: Persistent file storage for skills
- **Access**: Skills can read/write files in container

### ❌ Not Implemented (By Design)

- **MCP Servers**: Desktop app feature, not API-compatible
- **Persistent Memory Tool**: Requires complex client-side handlers; use document attachments instead

---

## Valve Configuration

### Admin Valves (`Valves` class)

#### API & Core Settings

```python
ANTHROPIC_API_KEY: str = Field(
    default="",
    description="Your Anthropic API key"
)

DEFAULT_MAX_TOKENS: int = Field(
    default=8192,
    description="Default maximum tokens for responses (max 8192)"
)

DEFAULT_TEMPERATURE: float = Field(
    default=1.0,
    description="Default temperature (0.0-1.0)"
)

REQUEST_TIMEOUT: int = Field(
    default=300,
    description="API request timeout in seconds"
)
```

#### Extended Thinking

```python
ENABLE_EXTENDED_THINKING: bool = Field(
    default=True,
    description="Enable extended thinking for complex reasoning"
)

THINKING_BUDGET_TOKENS: int = Field(
    default=10000,
    description="Thinking budget in tokens (1024-16000 recommended)"
)
```

#### Prompt Caching

```python
ENABLE_PROMPT_CACHING: bool = Field(
    default=True,
    description="Enable prompt caching (reduces cost up to 90%)"
)

CACHE_TTL: Literal["5min", "1hour"] = Field(
    default="5min",
    description="Cache duration: 5min (cheaper) or 1hour (longer sessions)"
)

CACHE_SYSTEM_PROMPT: bool = Field(
    default=True,
    description="Automatically cache system prompts"
)

CACHE_USER_MESSAGES: bool = Field(
    default=True,
    description="Cache recent user messages for stable cache points"
)
```

#### Web Search

```python
ENABLE_WEB_SEARCH: bool = Field(
    default=True,
    description="Enable web search capability"
)

WEB_SEARCH_MAX_USES: int = Field(
    default=5,
    description="Maximum web searches per request (1-20)"
)

WEB_SEARCH_DOMAIN_ALLOWLIST: str = Field(
    default="",
    description="Allowed domains (comma-separated, e.g., 'wikipedia.org,github.com')"
)

WEB_SEARCH_DOMAIN_BLOCKLIST: str = Field(
    default="",
    description="Blocked domains (comma-separated)"
)
```

#### Code Execution & Skills

```python
ENABLE_CODE_EXECUTION: bool = Field(
    default=False,
    description="Enable Python code execution (required for skills)"
)

ENABLE_SKILL_XLSX: bool = Field(
    default=True,
    description="Enable Excel spreadsheet support"
)

ENABLE_SKILL_PPTX: bool = Field(
    default=True,
    description="Enable PowerPoint support"
)

ENABLE_SKILL_DOCX: bool = Field(
    default=True,
    description="Enable Word document support"
)

ENABLE_SKILL_PDF: bool = Field(
    default=True,
    description="Enable PDF generation support"
)

CUSTOM_SKILL_IDS: str = Field(
    default="",
    description="Comma-separated custom skill IDs from your organization"
)
```

#### UX Settings

```python
SHOW_THINKING_PROCESS: bool = Field(
    default=True,
    description="Display thinking process in UI"
)

SHOW_WEB_SEARCH_DETAILS: bool = Field(
    default=True,
    description="Show full web search results with queries and snippets"
)

SHOW_CITATIONS: bool = Field(
    default=True,
    description="Show citation cards inline"
)

SHOW_TOKEN_USAGE: bool = Field(
    default=True,
    description="Display token usage statistics"
)

SHOW_PROCESSING_STATUS: bool = Field(
    default=True,
    description="Show status updates during processing"
)
```

### User Valves (`UserValves` class)

```python
THINKING_DISPLAY_MODE: Literal["expanded", "collapsed", "hidden"] = Field(
    default="expanded",
    description="How to display thinking: expanded (open by default), collapsed (closed), or hidden"
)

ENABLE_MY_WEB_SEARCH: bool = Field(
    default=True,
    description="Enable web search for my queries"
)

ENABLE_MY_CODE_EXECUTION: bool = Field(
    default=False,
    description="Enable code execution for my queries"
)
```

---

## Streaming Implementation

### State Management

Track multiple concurrent state machines:

```python
@dataclass
class StreamingState:
    thinking_state: ThinkingState  # NOT_STARTED, IN_PROGRESS, COMPLETED
    thinking_buffer: str = ""
    response_buffer: str = ""
    web_searches: List[WebSearchResult] = field(default_factory=list)
    citations: List[CitationData] = field(default_factory=list)
    current_block_index: int = 0
    current_block_type: Optional[str] = None
```

### Progressive Disclosure Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. THINKING STARTS                                          │
│    → Stream thinking text directly (visible, not collapsed) │
│    → User sees: "Let me analyze this problem..."            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. THINKING CONTINUES                                       │
│    → Continue streaming thinking deltas                     │
│    → Accumulate in thinking_buffer                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. WEB SEARCH TRIGGERED (if applicable)                     │
│    → Detect: server_tool_use with name="web_search"         │
│    → Emit status: "🔍 Searching: [query]"                   │
│    → Capture: web_search_tool_result block                  │
│    → Format: Rich card with query, titles, URLs, snippets   │
│    → Append search card to thinking stream                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. THINKING COMPLETES, RESPONSE STARTS                      │
│    → Detect: content_block_start with type="text"           │
│    → Action: Wrap thinking + searches in <details open>     │
│    → Yield complete thinking section                        │
│    → Switch to streaming response text                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. RESPONSE STREAMS                                         │
│    → Stream text_delta events normally                      │
│    → Collect citations from text blocks                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. RESPONSE COMPLETES                                       │
│    → Format citation cards inline                           │
│    → Show token usage stats                                 │
│    → Emit final citations via event emitter (optional)      │
└─────────────────────────────────────────────────────────────┘
```

### Event Processing Logic

```python
async def stream_response(self, ...):
    state = StreamingState()

    for line in response.iter_lines():
        if not line.startswith(b"data: "):
            continue

        data = json.loads(line[6:])
        event_type = data.get("type")

        if event_type == "content_block_start":
            content_block = data.get("content_block", {})
            block_type = content_block.get("type")
            state.current_block_type = block_type

            if block_type == "thinking":
                state.thinking_state = ThinkingState.IN_PROGRESS

            elif block_type == "text":
                # Thinking just finished, response starting
                if state.thinking_state == ThinkingState.IN_PROGRESS:
                    # Wrap thinking in <details>
                    yield self._format_thinking_section(state)
                    state.thinking_state = ThinkingState.COMPLETED

            elif block_type == "server_tool_use":
                # Web search starting
                if content_block.get("name") == "web_search":
                    query = content_block.get("input", {}).get("query", "")
                    await self.emit_status(__event_emitter__, f"🔍 Searching: {query}")

        elif event_type == "content_block_delta":
            delta = data.get("delta", {})
            delta_type = delta.get("type")

            if delta_type == "thinking_delta":
                thinking_text = delta.get("thinking", "")
                state.thinking_buffer += thinking_text
                yield thinking_text  # Stream thinking live

            elif delta_type == "text_delta":
                text = delta.get("text", "")
                yield text  # Stream response text

        elif event_type == "content_block_stop":
            # Check if this was a web_search_tool_result block
            if state.current_block_type == "web_search_tool_result":
                # Format and append search results to thinking
                search_card = self._format_web_search_results(...)
                yield search_card
```

### Formatting Functions

#### Thinking Section

```python
def _format_thinking_section(self, state: StreamingState) -> str:
    mode = self.user_valves.THINKING_DISPLAY_MODE

    if mode == "hidden" or not self.valves.SHOW_THINKING_PROCESS:
        return ""

    is_open = "open" if mode == "expanded" else ""

    output = f"\n\n<details {is_open}>\n"
    output += "<summary>🧠 Claude's Reasoning Process</summary>\n\n"
    output += state.thinking_buffer

    # Append web search results if any
    for search in state.web_searches:
        output += "\n\n" + self._format_web_search_card(search)

    output += "\n\n</details>\n\n"
    return output
```

#### Web Search Card

```python
def _format_web_search_card(self, search: WebSearchResult) -> str:
    if not self.valves.SHOW_WEB_SEARCH_DETAILS:
        return f"🔍 **Searched:** {search.query}\n\n---"

    output = f"🔍 **Web Search: \"{search.query}\"**\n\n"
    output += "**Results Found:**\n\n"

    for i, result in enumerate(search.results, 1):
        output += f"{i}. **{result.title}**\n"
        output += f"   🔗 {result.url}\n"
        if result.snippet:
            output += f"   > {result.snippet}\n\n"

    output += "---"
    return output
```

#### Citation Card

```python
def _format_citation_card(self, citation: CitationData, index: int) -> str:
    if not self.valves.SHOW_CITATIONS:
        return ""

    output = "\n---\n"
    output += f"📚 **Source [{index}]: {citation.title}**\n"
    output += f"🔗 {citation.url}\n"

    if citation.cited_text:
        excerpt = citation.cited_text[:150]
        if len(citation.cited_text) > 150:
            excerpt += "..."
        output += f"📝 \"{excerpt}\"\n"

    output += "---\n"
    return output
```

#### Token Usage

```python
def _format_token_usage(self, usage: Dict[str, Any]) -> str:
    if not self.valves.SHOW_TOKEN_USAGE or not usage:
        return ""

    output = "\n\n<details>\n<summary>📊 Token Usage</summary>\n\n"

    # Cache statistics
    cache_read = usage.get("cache_read_input_tokens", 0)
    cache_created = usage.get("cache_creation_input_tokens", 0)

    if cache_read > 0:
        total_input = usage.get("input_tokens", 0) + cache_read
        savings_pct = (cache_read / total_input * 100) if total_input > 0 else 0
        output += f"- 💾 **Cache Hit**: {cache_read:,} tokens ({savings_pct:.0f}% saved)\n"

    if cache_created > 0:
        output += f"- 📝 **Cached**: {cache_created:,} tokens\n"

    # Token counts
    thinking_tokens = usage.get("thinking_tokens", 0)
    if thinking_tokens > 0:
        output += f"- 🧠 **Thinking**: {thinking_tokens:,} tokens\n"

    output += f"- 📥 **Input**: {usage.get('input_tokens', 0):,} tokens\n"
    output += f"- 📤 **Output**: {usage.get('output_tokens', 0):,} tokens\n"

    # Tool usage
    server_tool_use = usage.get("server_tool_use", {})
    web_search_count = server_tool_use.get("web_search_requests", 0)
    if web_search_count > 0:
        output += f"- 🔍 **Web Searches**: {web_search_count}\n"

    output += "\n</details>\n"
    return output
```

---

## API Request Construction

### Beta Header Composition

```python
def _get_headers(self, user_valves) -> Dict[str, str]:
    """Dynamically compose beta headers based on enabled features."""
    betas = []

    # Prompt caching
    if self.valves.ENABLE_PROMPT_CACHING:
        betas.append("prompt-caching-2024-07-31")
        if self.valves.CACHE_TTL == "1hour":
            betas.append("extended-cache-ttl-2025-04-11")

    # Web search
    if self.valves.ENABLE_WEB_SEARCH and user_valves.ENABLE_MY_WEB_SEARCH:
        betas.append("web-search-2025-03-05")

    # Code execution + skills + files
    if self._should_enable_code_execution(user_valves):
        betas.append("code-execution-2025-08-25")
        betas.append("skills-2025-10-02")
        betas.append("files-api-2025-04-14")

    # Interleaved thinking (only when thinking + tools both enabled)
    if self.valves.ENABLE_EXTENDED_THINKING and self._has_tools(user_valves):
        betas.append("interleaved-thinking-2025-05-14")

    headers = {
        "x-api-key": self.valves.ANTHROPIC_API_KEY,
        "anthropic-version": "2023-06-01",
        "content-type": "application/json"
    }

    if betas:
        headers["anthropic-beta"] = ",".join(betas)

    return headers
```

### Helper Methods

```python
def _should_enable_code_execution(self, user_valves) -> bool:
    """Check if code execution should be enabled."""
    if not user_valves.ENABLE_MY_CODE_EXECUTION:
        return False

    if not self.valves.ENABLE_CODE_EXECUTION:
        return False

    # Check if any skills are enabled
    has_skills = (
        self.valves.ENABLE_SKILL_XLSX or
        self.valves.ENABLE_SKILL_PPTX or
        self.valves.ENABLE_SKILL_DOCX or
        self.valves.ENABLE_SKILL_PDF or
        bool(self.valves.CUSTOM_SKILL_IDS.strip())
    )

    return has_skills or self.valves.ENABLE_CODE_EXECUTION

def _has_tools(self, user_valves) -> bool:
    """Check if any tools are enabled."""
    return (
        (self.valves.ENABLE_WEB_SEARCH and user_valves.ENABLE_MY_WEB_SEARCH) or
        self._should_enable_code_execution(user_valves)
    )
```

### Tools Array Construction

```python
def _configure_tools(self, user_valves) -> List[Dict[str, Any]]:
    """Build the tools array for the API request."""
    tools = []

    # Web search tool
    if self.valves.ENABLE_WEB_SEARCH and user_valves.ENABLE_MY_WEB_SEARCH:
        tool = {
            "type": "web_search_20250305",
            "name": "web_search",
            "max_uses": min(max(1, self.valves.WEB_SEARCH_MAX_USES), 20)
        }

        # Domain filtering (cannot use both allow and block lists)
        if self.valves.WEB_SEARCH_DOMAIN_ALLOWLIST:
            domains = [d.strip() for d in self.valves.WEB_SEARCH_DOMAIN_ALLOWLIST.split(",") if d.strip()]
            if domains:
                tool["allowed_domains"] = domains
        elif self.valves.WEB_SEARCH_DOMAIN_BLOCKLIST:
            domains = [d.strip() for d in self.valves.WEB_SEARCH_DOMAIN_BLOCKLIST.split(",") if d.strip()]
            if domains:
                tool["blocked_domains"] = domains

        tools.append(tool)

    # Code execution with skills
    if self._should_enable_code_execution(user_valves):
        skill_ids = []

        # Add pre-built skills
        if self.valves.ENABLE_SKILL_XLSX:
            skill_ids.append("xlsx")
        if self.valves.ENABLE_SKILL_PPTX:
            skill_ids.append("pptx")
        if self.valves.ENABLE_SKILL_DOCX:
            skill_ids.append("docx")
        if self.valves.ENABLE_SKILL_PDF:
            skill_ids.append("pdf")

        # Add custom skills
        if self.valves.CUSTOM_SKILL_IDS:
            custom = [s.strip() for s in self.valves.CUSTOM_SKILL_IDS.split(",") if s.strip()]
            skill_ids.extend(custom)

        tool = {
            "type": "code_execution_20250825",
            "name": "code_execution"
        }

        if skill_ids:
            tool["container"] = {"skill_ids": skill_ids}

        tools.append(tool)

    return tools
```

### Thinking Configuration

```python
def _configure_thinking(self) -> Optional[Dict[str, Any]]:
    """Configure extended thinking if enabled."""
    if not self.valves.ENABLE_EXTENDED_THINKING:
        return None

    # Clamp budget to recommended range
    budget = max(1024, min(self.valves.THINKING_BUDGET_TOKENS, 16000))

    return {
        "type": "enabled",
        "budget_tokens": budget
    }
```

### Max Tokens Calculation

```python
def _calculate_max_tokens(self, requested_max: int) -> int:
    """
    Calculate appropriate max_tokens accounting for thinking budget.

    With extended thinking: max_tokens must accommodate both thinking and response.
    Without thinking: just use the requested amount (capped at 8192).
    """
    if not self.valves.ENABLE_EXTENDED_THINKING:
        return min(requested_max, 8192)

    thinking_budget = max(1024, min(self.valves.THINKING_BUDGET_TOKENS, 16000))

    # Reserve at least 2000 tokens for the actual response
    min_required = thinking_budget + 2000

    # Use the larger of requested or minimum required, cap at 8192
    calculated = max(requested_max, min_required)
    return min(calculated, 8192)
```

### Caching Application

```python
def _apply_caching(self, payload: Dict[str, Any]) -> Dict[str, Any]:
    """
    Apply prompt caching breakpoints to system prompt and user messages.

    Caching strategy:
    1. Cache last item in system prompt array
    2. Cache 2nd-to-last user message (creates stable cache point)
    """
    if not self.valves.ENABLE_PROMPT_CACHING:
        return payload

    cache_control = {"type": "ephemeral"}

    # Cache system prompt
    if self.valves.CACHE_SYSTEM_PROMPT and "system" in payload:
        system_content = payload["system"]

        if isinstance(system_content, str):
            # Convert string to array with cache control
            payload["system"] = [{
                "type": "text",
                "text": system_content,
                "cache_control": cache_control
            }]
        elif isinstance(system_content, list) and len(system_content) > 0:
            # Add cache control to last item
            payload["system"][-1]["cache_control"] = cache_control

    # Cache user messages (2nd-to-last user message)
    if self.valves.CACHE_USER_MESSAGES:
        messages = payload.get("messages", [])
        user_message_indices = [i for i, m in enumerate(messages) if m["role"] == "user"]

        # Need at least 2 user messages to cache the 2nd-to-last
        if len(user_message_indices) >= 2:
            idx = user_message_indices[-2]
            content = messages[idx]["content"]

            if isinstance(content, list) and len(content) > 0:
                # Add cache control to last content block of this message
                messages[idx]["content"][-1]["cache_control"] = cache_control

    return payload
```

### Complete Payload Assembly

```python
def _prepare_payload(
    self,
    body: Dict[str, Any],
    processed_messages: List[Dict[str, Any]],
    system_message: Optional[str],
    user_valves
) -> Dict[str, Any]:
    """Assemble the complete API request payload."""

    # Start with required fields
    payload = {
        "model": "claude-sonnet-4-5-20250929",
        "messages": processed_messages,
        "max_tokens": self._calculate_max_tokens(body.get("max_tokens", self.valves.DEFAULT_MAX_TOKENS)),
        "stream": body.get("stream", True)
    }

    # Temperature
    if "temperature" in body:
        payload["temperature"] = body["temperature"]
    elif self.valves.DEFAULT_TEMPERATURE != 1.0:
        payload["temperature"] = self.valves.DEFAULT_TEMPERATURE

    # Other sampling parameters
    if "top_k" in body:
        payload["top_k"] = body["top_k"]
    if "top_p" in body:
        payload["top_p"] = body["top_p"]
    if "stop_sequences" in body or "stop" in body:
        payload["stop_sequences"] = body.get("stop_sequences", body.get("stop", []))

    # System message
    if system_message:
        payload["system"] = system_message

    # Extended thinking
    thinking_config = self._configure_thinking()
    if thinking_config:
        payload["thinking"] = thinking_config

    # Tools
    tools = self._configure_tools(user_valves)
    if tools:
        payload["tools"] = tools

    # Apply caching last (after all content is in place)
    payload = self._apply_caching(payload)

    return payload
```

---

## Error Handling

### Pre-Request Validation

```python
async def pipe(self, body: Dict[str, Any], __user__=None, __event_emitter__=None, __event_call__=None):
    # Validation checks
    if not self.valves.ANTHROPIC_API_KEY:
        return "Error: ANTHROPIC_API_KEY is not configured. Please add your API key in the valve settings."

    if "messages" not in body or not body["messages"]:
        return "Error: No messages in request"

    # Check for conflicting domain filters
    if self.valves.WEB_SEARCH_DOMAIN_ALLOWLIST and self.valves.WEB_SEARCH_DOMAIN_BLOCKLIST:
        return "Error: Cannot use both allowed_domains and blocked_domains. Please use only one."
```

### Streaming Error Handling

```python
try:
    response = requests.post(url, headers=headers, json=payload, stream=True, timeout=(30, self.valves.REQUEST_TIMEOUT))

    if response.status_code != 200:
        error_detail = response.text
        if response.status_code == 429:
            yield "⚠️ **Rate limit exceeded**. Please wait a moment and try again."
        elif response.status_code == 401:
            yield "❌ **Authentication failed**. Please check your API key."
        elif response.status_code == 400:
            yield f"❌ **Bad request**: {error_detail}"
        else:
            yield f"❌ **API Error ({response.status_code})**: {error_detail}"
        return

    for line in response.iter_lines():
        try:
            # Process line
            ...
        except json.JSONDecodeError:
            logger.warning(f"Failed to parse JSON: {line}")
            continue  # Skip malformed events
        except Exception as e:
            logger.error(f"Error processing event: {e}")
            continue  # Continue processing other events

except requests.exceptions.Timeout:
    yield "\n\n⏱️ **Request timed out**. Partial response may be shown above."
except requests.exceptions.ConnectionError:
    yield "\n\n🔌 **Connection error**. Please check your internet connection."
except Exception as e:
    logger.error(f"Unexpected error: {e}", exc_info=True)
    yield f"\n\n❌ **Unexpected error**: {str(e)}"
```

### Tool-Specific Error Handling

```python
# Web search graceful degradation
if event_type == "error" and "web_search" in str(data):
    logger.warning(f"Web search error: {data}")
    yield "\n\n⚠️ *Web search temporarily unavailable, continuing without search results...*\n\n"
    continue

# Skill not found
if event_type == "error" and "skill" in str(data).lower():
    skill_id = extract_skill_id(data)
    logger.warning(f"Skill '{skill_id}' not found: {data}")
    yield f"\n\n⚠️ *Skill '{skill_id}' not found. Continuing without this skill...*\n\n"
    continue
```

---

## Bug Fixes from v2.0.3

### Critical Bug #1: Citation Extraction

**Problem**: Current code looks for citations in `message_start` events, but citations actually appear in text content blocks.

**Current (Broken) Code**:
```python
if event.get("type") == "message_start":
    message = event.get("message", {})
    content = message.get("content", [])
    for block in content:
        if "citations" in block:  # This never happens!
            ...
```

**Fixed Code**:
```python
# Citations are in text blocks during streaming
if event_type == "content_block_start":
    content_block = data.get("content_block", {})
    if content_block.get("type") == "text" and "citations" in content_block:
        for cit in content_block["citations"]:
            citations.append(CitationData(
                url=cit.get("url", ""),
                title=cit.get("title", ""),
                cited_text=cit.get("cited_text", "")
            ))

# Also collect from final message in non-streaming
if not streaming:
    for block in response_data.get("content", []):
        if block.get("type") == "text" and "citations" in block:
            for cit in block["citations"]:
                citations.append(...)
```

### Critical Bug #2: Missing Citation Flag

**Problem**: OpenWebUI requires `self.citation = False` to use custom citation formatting.

**Fix**:
```python
def __init__(self):
    self.type = "manifold"
    self.id = "claude_complete"
    self.name = "claude/"
    self.citation = False  # REQUIRED: Disable OpenWebUI auto-citations
    ...
```

### Critical Bug #3: Thinking State Management

**Problem**: Current code doesn't properly track when thinking transitions to response.

**Fix**: Use explicit state machine with three states and proper transitions:
```python
class ThinkingState(Enum):
    NOT_STARTED = 0
    IN_PROGRESS = 1
    COMPLETED = 2

# Transition logic
if block_type == "thinking":
    thinking_state = ThinkingState.IN_PROGRESS
elif block_type == "text":
    if thinking_state == ThinkingState.IN_PROGRESS:
        # Wrap thinking, emit it, then start response
        yield format_thinking_section()
        thinking_state = ThinkingState.COMPLETED
    yield text_delta
```

---

## Implementation Checklist

### Phase 1: Core Structure ✓
- [ ] Create Pipe class with `self.citation = False`
- [ ] Define Valves and UserValves with all fields
- [ ] Implement `pipes()` method returning model list
- [ ] Implement `__init__` with proper initialization

### Phase 2: API Request Construction ✓
- [ ] Implement `_get_headers()` with dynamic beta composition
- [ ] Implement `_configure_tools()` for web search + code execution
- [ ] Implement `_configure_thinking()` for extended thinking
- [ ] Implement `_apply_caching()` with correct breakpoints
- [ ] Implement `_calculate_max_tokens()` accounting for thinking
- [ ] Implement `_prepare_payload()` for complete request assembly
- [ ] Implement message processing (images, PDFs, text)

### Phase 3: Streaming Implementation ✓
- [ ] Create StreamingState dataclass
- [ ] Implement main streaming loop with event parsing
- [ ] Implement thinking state machine (NOT_STARTED → IN_PROGRESS → COMPLETED)
- [ ] Implement web search detection and formatting
- [ ] Implement citation extraction from text blocks
- [ ] Implement progressive disclosure (thinking → response)
- [ ] Implement `_format_thinking_section()`
- [ ] Implement `_format_web_search_card()`
- [ ] Implement `_format_citation_card()`
- [ ] Implement `_format_token_usage()`

### Phase 4: Error Handling ✓
- [ ] Add pre-request validation
- [ ] Add HTTP status code handling (401, 429, 400, 500)
- [ ] Add timeout handling with partial response
- [ ] Add malformed JSON event handling
- [ ] Add graceful degradation for tool failures
- [ ] Add logging throughout

### Phase 5: Testing ✓
- [ ] Test basic query without thinking
- [ ] Test extended thinking with progressive disclosure
- [ ] Test web search with citation extraction
- [ ] Test skills (xlsx, pptx, docx, pdf)
- [ ] Test prompt caching (verify cache hit stats)
- [ ] Test domain filtering (allow/block lists)
- [ ] Test error scenarios (bad API key, timeout, etc.)
- [ ] Test user valve overrides
- [ ] Test non-streaming mode

### Phase 6: Documentation ✓
- [ ] Update function docstring with features list
- [ ] Add usage examples in comments
- [ ] Document valve options
- [ ] Add troubleshooting guide
- [ ] Update version to 3.0.0

---

## Success Criteria

The implementation will be considered successful when:

1. **Citations Work**: Web search citations appear as rich embedded cards
2. **Thinking Streams**: Thinking text appears progressively, then collapses when response starts
3. **Web Search Details**: Full query, titles, URLs, and snippets are shown
4. **Skills Function**: Pre-built and custom skills work correctly
5. **Caching Saves Cost**: Cache hit statistics show up to 90% savings on repeated content
6. **Error Handling**: Graceful degradation for all error scenarios
7. **User Control**: All valves work as expected for admin and per-user configuration

---

## Future Enhancements (v4.0+)

Potential features for future versions:

- **Persistent Memory**: Implement client-side file handlers for memory tool
- **MCP Integration**: Build wrapper to connect local MCP servers
- **Search Result Content Blocks**: Use newer citations API for RAG applications
- **Batch API Support**: For higher thinking budgets (>32k tokens)
- **Multi-turn Search**: Track search context across conversation
- **Cost Tracking**: Per-user cost monitoring and limits
- **A/B Testing**: Compare thinking vs non-thinking quality

---

## Appendix: API Reference Quick Links

- **Extended Thinking**: https://docs.claude.com/en/docs/build-with-claude/extended-thinking
- **Prompt Caching**: https://docs.claude.com/en/docs/build-with-claude/prompt-caching
- **Web Search**: https://docs.claude.com/en/docs/build-with-claude/tool-use/web-search-tool
- **Skills**: https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview
- **Code Execution**: https://docs.claude.com/en/docs/agents-and-tools/tool-use/code-execution
- **OpenWebUI Events**: https://docs.openwebui.com/features/plugin/events/
- **OpenWebUI Pipes**: https://docs.openwebui.com/features/plugin/functions/pipe/

---

**End of Design Document**
