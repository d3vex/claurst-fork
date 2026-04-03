# Message Copy Implementation Report

## Overview
Implemented granular message copy options in the Rust Claude Code port with 5 different copy formats accessible through context menu and keyboard interactions.

## Files Modified/Created

### 1. **crates/tui/src/message_copy.rs** (NEW)
Complete module for message formatting and clipboard integration.

**Key Functions:**
- `copy_as_markdown(message)` - Preserves all markdown formatting
- `copy_as_plaintext(message)` - Strips all markdown syntax
- `copy_code_blocks(message)` - Extracts only code blocks
- `copy_as_json(message)` - Serializes message as JSON
- `copy_to_clipboard(text)` - Cross-platform clipboard access (Windows, macOS, Linux)

**Features:**
- Handles structured message content (text blocks, tool use, tool results, thinking blocks)
- Markdown stripping removes: bold, italic, headers, links, blockquotes, inline code
- Code block extraction: identifies triple-backtick blocks, preserves language tags
- JSON serialization: includes role, uuid, cost information
- Cross-platform clipboard using native CLI tools (clip.exe, pbcopy, xclip, xsel)

### 2. **crates/tui/src/app.rs** (MODIFIED)
Enhanced context menu with copy variants.

**Changes:**
- Extended `ContextMenuItem` enum with:
  - `CopyAsMarkdown`
  - `CopyAsPlaintext`
  - `CopyCodeBlocks`
  - `CopyAsJson`
- Updated `navigate_context_menu()` to handle 9 items instead of 5
- Updated `execute_context_menu_item()` to include new variants
- Implemented `handle_context_menu_action()` handlers for all copy variants
- Integrated with `NotificationQueue` for user feedback

### 3. **crates/tui/src/render.rs** (MODIFIED)
Updated context menu rendering to show new copy options.

**Changes:**
- Expanded menu items from 5 to 9:
  1. Copy
  2. Copy as MD (Markdown)
  3. Copy Plain (Plaintext)
  4. Copy Code (Code blocks)
  5. Copy JSON
  6. Paste
  7. Cut
  8. Select All
  9. Clear
- Increased menu width from 15 to 16 characters
- Updated menu height constraint from 10 to 15 items max

### 4. **crates/tui/src/lib.rs** (MODIFIED)
Added public module declaration for message_copy.

## Copy Formats

### 1. Copy (Selection)
- Copies currently selected text from message pane
- Uses existing selection mechanism

### 2. Copy as Markdown
- Preserves all markdown syntax
- Formats tool use as JSON code blocks
- Thinking blocks as HTML details elements
- Example output:
  ```
  **Assistant**

  Here is the **bold** text with [links](url).
  ```

### 3. Copy as Plaintext
- Removes all markdown formatting
- Preserves content structure
- Example output:
  ```
  Assistant:

  Here is the bold text with links.
  ```

### 4. Copy Code Blocks
- Extracts all triple-backtick code blocks
- Preserves language identifiers
- Concatenates multiple blocks with separator
- Example output:
  ```
  ```rust
  fn main() {}
  ```

  ---

  ```python
  print("hello")
  ```

### 5. Copy as JSON
- Serializes complete message structure
- Includes role, content, uuid, and cost information
- Example output:
  ```json
  {
    "role": "assistant",
    "content": "Message text here...",
    "uuid": "12345...",
    "cost": {
      "input_tokens": 100,
      "output_tokens": 50,
      ...
    }
  }
  ```

## Clipboard Integration

Cross-platform clipboard support:

**Windows:**
- Uses `clip.exe` via stdin pipe

**macOS:**
- Uses `pbcopy` via stdin pipe

**Linux:**
- Tries `xclip -selection clipboard` first
- Falls back to `xsel --clipboard --input`

All implementations write via stdin to avoid shell escaping issues.

## User Feedback

Each copy action shows a notification:
- Success: "Copied as Markdown." (3-second duration)
- Failure: "Failed to copy to clipboard." (warning level, 3-second duration)

Notifications integrate with existing notification queue system.

## Context Menu Behavior

Right-click in message pane shows expanded context menu with:
- Arrow keys to navigate all 9 items
- Enter to select and execute
- Item enabling logic:
  - Copy/Cut: only enabled if text selected
  - Copy variants: only enabled if messages exist
  - Paste/SelectAll: always enabled
  - Clear: only enabled if selection exists

## Technical Implementation Details

### Message Structure Handling
- Supports both `MessageContent::Text` and `MessageContent::Blocks`
- Handles complex content blocks:
  - Text blocks
  - Tool use blocks (with JSON input)
  - Tool results (text or blocks)
  - Thinking blocks (with signature)
  - Image blocks (placeholder)

### Markdown Stripping Algorithm
- Character-by-character parsing with lookahead
- Handles:
  - Bold/italic markers (* and _)
  - Inline code (`)
  - Links [text](url) → text
  - Images ![alt](url) → skipped
  - Headers (# ## ###) → stripped prefix only
  - Blockquotes (>) → stripped prefix

### Code Block Extraction
- Line-by-line parsing
- Identifies ````lang` boundaries
- Skips language identifier in output
- Handles unclosed blocks gracefully

## Testing Recommendations

1. **Copy Selection**: Select text with mouse, right-click → Copy
2. **Markdown Format**: Right-click → Copy as MD, verify markdown preserved
3. **Plaintext Format**: Right-click → Copy Plain, verify markdown stripped
4. **Code Extraction**: Send message with code blocks, Copy Code should extract all
5. **JSON Export**: Right-click → Copy JSON, verify complete structure
6. **Clipboard Fallback**: Test on system without clipboard access (verify graceful failure)

## Compilation Status

✓ No compilation errors in message_copy module
✓ Integrates cleanly with existing app.rs
✓ Render.rs changes are minimal and backward compatible
✓ All function signatures match Message type requirements

## Future Enhancements

Potential improvements:
1. Add keyboard shortcuts (e.g., Ctrl+Shift+C for copy format dialog)
2. Remember last used copy format as default
3. Add "Copy with timestamp" format
4. Rich text clipboard support (RTF/HTML) on Windows
5. Copy formatting preferences in settings
6. Batch copy multiple messages
