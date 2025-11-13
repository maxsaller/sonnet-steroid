# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [4.0.0] - 2025-11-13

### Added
- ✅ **`<think>` tag support** for collapsible, real-time thinking display
  - OpenWebUI-native collapsible thinking sections
  - Real-time streaming of reasoning as it happens
  - Clickable "Thinking..." notification
- ✅ **JSON fragment accumulation** for web search query capture
  - Accumulates partial JSON fragments across streaming events
  - Robust parsing of character-by-character query streaming
  - Fixes "query not captured" issue from v3
- ✅ **Clean UI design** without icons
  - Removed icons from collapsible sections
  - Professional, minimal appearance
  - Markdown-only formatting (no HTML/CSS)
- ✅ **Comprehensive logging** for debugging
  - Detailed event logging
  - Query accumulation trace logs
  - Token usage tracking

### Fixed
- 🐛 **Opening `<think>` tag not appearing** (critical bug)
  - Root cause: State set too early in `content_block_start`
  - Solution: Let first `thinking_delta` handle state and tag opening
  - Result: Thinking now displays correctly in collapsible section
- 🐛 **Web search queries showing as "(query not captured)"**
  - Root cause: Queries stream as character fragments, not complete JSON
  - Solution: Accumulate fragments in buffer until valid JSON
  - Result: All search queries now captured and displayed correctly
- 🐛 **Citation chips working correctly**
  - Fixed event emitter format for OpenWebUI
  - Web search results properly formatted as citations
  - Clickable citation chips at response bottom

### Changed
- 📝 **Switched from HTML/CSS to Markdown** for all formatting
  - OpenWebUI doesn't render inline HTML
  - All sections now use Markdown formatting
  - Collapsible sections use `<details>` tags
- 📝 **Removed reasoning section at bottom**
  - Redundant with `<think>` tags
  - Cleaner response format
  - Thinking integrated into response flow

### Technical Details
- **Think tag state machine:**
  ```python
  NOT_STARTED → first thinking_delta opens <think> → IN_PROGRESS
  → content_block_stop closes </think> → COMPLETED
  ```
- **Query accumulation algorithm:**
  ```python
  partial_json_buffer += fragment
  try: json.loads(buffer) → extract query
  except: continue accumulating
  ```

### Migration from v3
- Replace `claude_sonnet_complete_v3.py` with `function.py`
- All valve settings preserved
- Improved UX with no configuration changes needed

---

## [3.0.0] - 2025-11-12

### Added
- ✅ **Extended thinking** with configurable budget (1,024-16,000 tokens)
- ✅ **Web search** powered by Anthropic API
- ✅ **Prompt caching** with 5min/1hour TTL options
- ✅ **Citations** from web search results
- ✅ **Skills system** (xlsx, pptx, docx, pdf) with custom skill support
- ✅ **Code execution** in secure sandbox
- ✅ **Dynamic beta header composition**
- ✅ **Collapsible sections** for web searches and token usage
- ✅ **Comprehensive valve system** for admin and user settings

### Issues (Deprecated)
- ❌ **Thinking displayed as raw text** instead of collapsible
  - Attempted HTML/CSS formatting, but OpenWebUI doesn't render inline HTML
  - Progress bar notifications stacked incorrectly
  - No real-time streaming of thinking text
- ❌ **Web search query capture broken**
  - Queries streaming as character fragments not handled
  - All searches showed "(query not captured)"
  - No accumulation buffer implemented
- ❌ **HTML/CSS rendered as raw text**
  - Beautiful cards and formatting displayed as `<div style='...'>`
  - OpenWebUI treats output as Markdown, escapes HTML
- ❌ **Citations inconsistent**
  - Formal API citations often empty
  - Fallback citation implementation incomplete

### Deprecation Notice
**v3.0.0 is deprecated** and replaced by v4.0.0. See [`archive/v3.0.0/function.py`](archive/v3.0.0/function.py) for historical reference.

---

## Version Comparison

| Feature | v3.0.0 | v4.0.0 |
|---------|--------|--------|
| Extended thinking | ⚠️ Raw text | ✅ Collapsible `<think>` |
| Web search queries | ❌ Not captured | ✅ Captured correctly |
| UI formatting | ❌ HTML (broken) | ✅ Markdown |
| Citations | ⚠️ Inconsistent | ✅ Working |
| Thinking streaming | ❌ Progress bar | ✅ Real-time text |
| Clean UI | ❌ Icons, HTML | ✅ Minimal design |
| Debugging | ⚠️ Basic logs | ✅ Comprehensive |

---

## Future Roadmap

### Planned Features
- 📄 **Document citations with inline markers** - `[1]`, `[2]` in response text
  - Waiting for Anthropic to add character positions to web search citations
  - Currently possible with uploaded documents (PDF, text)
- 🔧 **Advanced caching strategies** - Per-conversation cache management
- 📊 **Usage analytics** - Built-in token/cost tracking dashboard
- 🎨 **Customizable UI themes** - User-selectable response formatting
- 🔌 **Webhook support** - External integrations and notifications

### Under Consideration
- 🌐 **Multi-model support** - Opus, Haiku alongside Sonnet
- 🔄 **Conversation branching** - Alternative response paths
- 📝 **Response templating** - Structured output formats
- 🛡️ **Enhanced security** - Additional sandboxing options

---

## Breaking Changes

### v3.0.0 → v4.0.0
- **No breaking changes to valves** - All settings preserved
- **UI changes only** - Better UX, same functionality
- **Drop-in replacement** - Copy new `function.py`, done!

---

## Known Issues

### Current Limitations
1. **Web search citations don't include inline markers** - By design (API limitation)
2. **Thinking budget is a maximum, not a guarantee** - Actual usage varies
3. **Cache TTL cannot be customized per-request** - Global setting only

See [Troubleshooting Guide](docs/guides/troubleshooting.md) for solutions to common issues.

---

## Development History

### Design Phase (2025-11-12)
- Created comprehensive design document
- Researched Claude API features
- Planned valve system architecture
- Defined event processing pipeline

### v3 Implementation (2025-11-12)
- Implemented core features
- Attempted HTML/CSS formatting
- Encountered OpenWebUI rendering limitations
- Identified critical bugs

### v4 Implementation (2025-11-13)
- Adopted `<think>` tags for thinking display
- Implemented JSON fragment accumulation
- Fixed all critical bugs from v3
- Cleaned up UI design
- Added comprehensive logging

### Documentation Overhaul (2025-11-13)
- Restructured repository
- Created comprehensive guides
- Added API reference
- Documented architecture
- Created this changelog

---

## Contributing

Found a bug? Want to add a feature? See our [Contributing Guidelines](README.md#-contributing).

---

## Acknowledgments

**Technologies:**
- Anthropic Claude API and extended thinking capabilities
- OpenWebUI platform and community

---

**Questions?** Open an [issue](../../issues) or [discussion](../../discussions)!
