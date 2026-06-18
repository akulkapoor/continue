# Session Trace Workflow

This note summarizes the session trace work added across the PR series. The goal is to make Continue chat sessions easy to save, inspect, compare, and clean up from VS Code without changing the existing persisted session schema.

## Goals

- Save the current Continue chat session as a human-readable trace.
- View recently saved traces.
- Open a saved trace as Markdown.
- Diff two saved traces in VS Code.
- Clear saved trace files.

## PR Breakdown

1. `Add Markdown renderer for session traces`

   - Added core utilities for rendering a persisted `Session` as deterministic Markdown.
   - Standardized trace event types and tool call/result rendering.
   - Added unit coverage for trace formatting semantics.

2. `Add storage helpers for session trace files`

   - Added `~/.continue/session-traces` as the trace storage directory.
   - Added `.continue-trace.md` filename generation.
   - Added frontmatter metadata parsing and recent-first trace listing.
   - Added unit coverage for filenames, metadata parsing, and listing.

3. `Add VS Code command to save session traces`

   - Added `continue.saveSessionTrace`.
   - Loads the current persisted session via `getCurrentSessionId` and `history/load`.
   - Renders Markdown using the core trace renderer.
   - Saves the trace under `~/.continue/session-traces`.
   - Offers to open the saved trace.

4. `Add VS Code commands for managing session traces`

   - Added `continue.viewRecentSessionTraces`.
   - Added `continue.clearSessionTraces`.
   - Removed the duplicate `openSessionTrace` command because it behaved the same as viewing recent traces.
   - Made clear delete all `.continue-trace.md` files, including malformed traces.

5. `Add VS Code command to diff session traces`
   - Added `continue.diffSessionTraces`.
   - Lets the user pick two valid saved traces.
   - Opens VS Code's built-in Markdown diff view.

## Trace Format

Traces are Markdown files with YAML frontmatter:

```yaml
---
traceVersion: 1
sessionId: "..."
title: "..."
workspaceDirectory: "..."
traceCreatedAt: "..."
messageCount: 0
---
```

Events are rendered as stable numbered Markdown headings:

```md
## 001 user_message

## 002 assistant_message

## 003 tool_call: runTerminalCommand

## 004 tool_result: runTerminalCommand
```

V1 event types:

- `user_message`
- `assistant_message`
- `reasoning`
- `tool_call`
- `tool_result`
- `summary`

## Design Decisions

### Use Persisted Sessions Only

Trace generation reads the existing persisted `Session` JSON. It does not force-save or inspect in-memory GUI streaming state.

This means a trace includes completed/saved chat turns only. In-progress streaming content is intentionally excluded.

### Keep the Session Schema Unchanged

The trace format is derived from the existing `Session` shape. We did not add a new session schema or a new webview protocol request.

### Store Traces Separately

Saved traces live under:

```text
~/.continue/session-traces
```

This keeps trace snapshots separate from normal Continue chat history.

### Markdown Over Raw JSON

The trace snapshot is human-readable Markdown rather than raw session JSON. The intent is quick inspection, review, and diffing.

Raw session export was considered separately, but the trace workflow focuses on readable Markdown.

### Tool Outcomes Belong in `tool_result`

Tool events are split into:

- `tool_call`: what the assistant requested.
- `tool_result`: what happened.

Success, failure, output, and error text are rendered in `tool_result` only. This avoids duplicating error/status text in both call and result events.

### Clear Deletes Malformed Trace Files

`View Recent Session Traces` lists valid traces with parseable metadata.

`Clear Saved Session Traces` deletes all files ending in `.continue-trace.md`, including malformed trace files. This lets clear act as cleanup even when a trace cannot appear in the recent-traces picker.

### Diff Uses VS Code's Built-In Diff

The diff command opens the two selected Markdown trace files with `vscode.diff`. We did not build a custom diff renderer.

## VS Code Commands

- `Continue: Save Current Chat Session Trace`

  - Saves the current persisted session as a `.continue-trace.md` file.

- `Continue: View Recent Session Traces`

  - Shows valid saved traces in a QuickPick and opens the selected Markdown file.

- `Continue: Clear Saved Session Traces`

  - Deletes saved trace files from `~/.continue/session-traces` after confirmation.

- `Continue: Diff Session Traces`
  - Lets the user pick two valid saved traces and opens a Markdown diff.

## Important Edge Cases

- Missing persisted session file:

  - Save reports a load failure instead of treating it as an empty chat.

- Empty persisted session:

  - Save reports that there are no saved chat messages.

- Streaming response:

  - Save includes only previously persisted completed messages.

- Malformed trace metadata:

  - View recent skips the malformed file.
  - Clear still deletes the malformed `.continue-trace.md` file.

- Filename and frontmatter timestamp:
  - Save creates one `Date` and passes it to both filename generation and Markdown rendering so the filename timestamp and `traceCreatedAt` stay paired.

## Manual Testing Checklist

1. Save a completed chat and confirm the trace opens as Markdown.
2. Save while a response is streaming and confirm only completed persisted messages are included.
3. Save a session with tool calls and confirm tool calls/results are readable.
4. View recent traces and open one from the QuickPick.
5. Try viewing recent traces with no valid saved traces.
6. Diff two saved traces and confirm VS Code opens a side-by-side diff.
7. Try diffing with fewer than two valid saved traces.
8. Clear saved traces and confirm `.continue-trace.md` files are deleted.
9. Confirm normal Continue chat history is not deleted by clearing traces.
