# Rewind a task's conversation to an earlier message (conversation only)

- Date: 2026-11-09
- Category: feature
- Priority signal: medium — a stray or wrong message permanently steers the agent today, and there is no way back short of starting a new task
- Risk signal: high — per `SDLC.md` it touches the HTTP API surface (an optional rewind field on `POST /runs/:id/continue`), the runner spec (`AgentRunSpec` gains a resume-at target) and the run event log (a new rewound marker); all additive, no file restore
- Routing: `om-prepare-issue "Rewind a task's conversation to an earlier message — brief: .ai/specs/briefs/2026-11-09-conversation-rewind.md" --no-spec`, then `om-spec-writing "Rewind a task's conversation to an earlier message — brief: .ai/specs/briefs/2026-11-09-conversation-rewind.md"` (issue and spec PR on the `kmrosiek/cezar` fork only)

## Problem

In a cezar task the user sometimes sends an unrelated message, checks something irrelevant, or makes a mistake. The agent then gets sidetracked — the next real instruction goes in an unexpected direction because the wrong message is still in its context — and the transcript gets cluttered. Plain Claude Code has `/rewind` (Esc Esc); cezar has nothing equivalent. Verified on `main` (bf656d01): a delivered message has no actions — edit/remove exist only while a task is queued (`packages/web/src/routes/task-thread/task-thread.tsx:288-319`) — and Continue always resumes the same provider session (`claude --resume <id>`, `packages/cezar/src/core/claude-cli-runner.ts:452-457`). No issue or PR on `open-mercato/cezar` proposes rewind; the nearest is #817 (start a fresh run carrying a prior session's history), which is a different capability.

## Agreed direction

A **Rewind** action on a sent user message, modelled on Claude Code's `/rewind`:

1. **Entry.** Desktop: the existing hover action row on the user bubble (`thread-items.tsx:242`). Mobile: long-press opens a menu — **Rewind**, **Copy**, **Select text** — reusing `packages/web/src/lib/use-long-press.ts`. Long-press no longer starts native text selection; "Select text" turns selection on for that message.
2. **Prefilled editor.** Rewind opens the message's text in the inline editor (the queued-message editor, with #939 draft persistence). The user edits it, replaces it entirely, or cancels.
3. **"What to restore" area.** In v1 it is only a notice: *"Conversation rewinds here · files stay as they are."* It deliberately does not list files — cezar sees the agent's edit-tool calls but not changes made through shell commands, so a list could be incomplete and misleadingly reassuring. This area is where a conversation / files / both picker goes later.
4. **Send applies it.** Nothing happens to the run until Send: the pending rewind lives in the client draft, not on the run record, so there is no half-rewound server state that other session writers (child reports, auto-resume nudges, landing retries) would have to honour.
5. **Mechanism (Claude).** Record the `uuid` of each turn's final assistant message from Claude's stream-json output (it matches the session file's id; cezar discards it today). On Send, the continuation runs `claude --resume <session> --resume-session-at <uuid of the reply before the chosen message>` with the edited text; later Continues are plain `--resume` and stay on the rewound branch.
6. **After.** The rewound messages collapse in the transcript behind *"↩ Rewound N messages · files unchanged · show"* (the NDJSON log is append-only — hide, never delete). The agent receives a short note that files were not reverted and to check `git status`.

**Rejected:** a standalone *Rewind then type* (needs a pending server-side cut state); a Claude.ai-style *Edit* label (a files option would not fit it later); *Retry*; a blocking pop-up confirmation on every rewind (the in-editor notice plus Send is the confirmation, and nothing is destroyed — the old branch stays in Claude's session file and cezar's log); restoring files in v1. "Build nothing" lost because the only workaround — "Open in…" → interactive `claude --resume` → `/rewind` — leaves cezar's transcript out of sync with the session (untested) and works only on a terminal.

## Resolved unknowns

| Question | Answer (from the conversation) |
|----------|--------------------------------|
| Does Claude support rewinding a headless session? | Yes — verified with Claude Code 2.1.319: `--resume <id> --resume-session-at <assistant uuid>` with `--print --input-format stream-json` makes the model forget everything after that reply; the session id is unchanged; a later plain `--resume` follows the rewound branch. |
| Does the context actually shrink? | Yes. The session file is a tree (`parentUuid`); context is the path from the newest message to the root, and the rewound branch is off that path. Spend already incurred stays spent. |
| What if the uuid is unknown? | The CLI exits 1 with "No message found with message.uuid of: …" — a loud failure, no silent full resume; the session is left intact. |
| What about compaction? | After a compaction the boundary entry has `parentUuid: null`; rewinding to a message **before** it fails with the same "No message found" error, rewinding to one **after** it works (verified with `/compact`). Disable Rewind on messages before the latest compaction, with a reason ("the agent only has a summary of earlier messages"). Cezar must start recording Claude's `system/compact_boundary` event — its Claude mapping ignores it today. |
| Files? | Conversation only. Worktree and in-place tasks behave the same: files stay as they are. (Turn-end autosave is not per message — it runs at session end, before check steps, dispatch and PRs — so there are no per-message file checkpoints to restore anyway.) |
| Which backends? | Claude only in v1; the action is hidden for other runners until they offer an equivalent. |
| When is it available? | Idle tasks only. Not on the first message (the task prompt — no reply before it). Not on messages from an earlier session (before an agent fallback/switch). Not before the latest compaction. |
| Retry button? | No. |
| Rewind without sending a message? | Not in v1. Later idea: emptying the editor offers "Rewind to this point". |
| Files notice content? | Generic only — no file names. |

## Non-goals

- Restoring worktree or in-place files (a later "what to restore" picker; candidates: Claude's hidden `--rewind-files`, or a git snapshot cezar records at each user message).
- Retry / regenerate.
- Branch navigation between rewound and current versions (Claude.ai's ‹ 1/2 ›).
- Rewinding non-Claude backends.
- Rewinding past a compaction, or starting a new task from a message (#817).
- Rewinding while the task is running.

## Affected areas (if known)

- `packages/cezar/src/core/claude-cli-runner.ts` — capture assistant `uuid`s; pass `--resume-session-at`; surface `compact_boundary`.
- `packages/cezar/src/workflows/run.ts` — `continueRun` / `runContinuation` accept a rewind target; inject the "files not reverted" note.
- `packages/cezar/src/core/agent-runner.ts:91-96` — `AgentRunSpec` (`sessionId`, `resume`) gains a resume-at target (runner seam).
- `packages/contract/src/runs.ts:1318` (`continueInputSchema`) + `POST /runs/:id/continue` — an optional rewind field.
- `packages/web/src/routes/task-thread/thread-items.tsx`, `task-thread.tsx`, `session-transcript.tsx` — action row, long-press menu, prefilled editor, rewound-messages marker.
- Verified 2026-11-10 (spec phase): rewinding to the final assistant message of a tool-using turn works and keeps the tool output; a mid-turn `tool_use` anchor silently loses the tool result — anchors are captured only at `result`.
