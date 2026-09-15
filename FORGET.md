# `/forget` — context rollback (fork addition)

This repository is a fork of [earendil-works/pi](https://github.com/earendil-works/pi) (MIT) that adds **one feature**: the **`/forget`** slash command, which removes the last N user turns from the model's context — optionally from the session file as well. Nothing else in this fork differs from upstream.

- **Upstream base:** `f9bcd351d` (v0.85.1)
- **Fork delta:** commits `3a15395d3..HEAD` on top of the base: the feature lives in `packages/coding-agent` (9 files), plus this file and the README banner
- **Upstream status:** not yet merged — [PR earendil-works/pi#9615](https://github.com/earendil-works/pi/pull/9615). The feature delta is small and self-contained; it can be ported anywhere near the base commit with `git format-patch f9bcd351d..HEAD` + `git am`.

## Why this feature exists

1. **The harness owns the model context, and pi had no rollback command.** In pi, what the model sees is derived from the session tree. When recent turns go wrong (a bad assumption, poisoned context, a direction you want to retry), the only options were `/new` (throw away everything, including the parts you want to keep) or hand-editing the session JSONL file. `/forget` makes context rollback a first-class, safe operation.
2. **Existing navigation cannot guarantee "the model has no awareness."** You can approximate soft forgetting with `/tree` + "no summary", but the `/tree` flow is designed to *offer* a branch summary of the path you leave — and one accepted summary re-introduces the "forgotten" content into the model context. `/forget` makes the no-summary path the only path, so the guarantee is structural rather than a matter of per-navigation discipline.
3. **The append-only session format cannot physically delete turns.** Session files are append-only by design and `/tree` only moves the leaf. When the goal is "these turns must not exist in the file" — not visible in `/tree`, exports, shares, or future resumes — a sanctioned rewrite is required. `--hard` provides it with explicit confirmation and, by design, no backup.

## What it does

| Mode | Command | Model context | Session file | Recoverable? |
|------|---------|---------------|--------------|--------------|
| Soft (default) | `/forget [N]` | Removed turns disappear | Unchanged — removed turns stay in the file | Yes, via `/tree` |
| Hard | `/forget [N] --hard` | Removed turns disappear | Rewritten to contain only the retained path | **No — no backup is written. This cannot be undone.** |

Key properties:

- **The model has no awareness of the removed turns.** No branch summary is created and no compaction summary is generated for the removed content. The next response is generated from exactly the retained messages. (The guarantee is conditional on later `/tree` use — see [Limitations](#limitations).)
- **The cut always lands on a user-message boundary.** A "user turn" is a user message plus everything after it up to the next user message (or the leaf) — including the assistant's `toolCall`/`toolResult` traffic. This is why a turn that used a tool removes 4 entries instead of 2: removing the user message and the final answer while leaving the tool call behind would (a) leave the model with direct evidence of the "forgotten" turn, and (b) leave a dangling `toolResult` with no matching `toolCall`, which is an invalid context.
- **Hard mode asks for confirmation**, stating: *"The session file will be rewritten. No backup will be written. This cannot be undone."* If the session has abandoned branches (from `/tree`), the dialog also states how many of their entries will be deleted.
- **`/forget` is only accepted while idle** — it is refused while a response, compaction, or tree navigation is in progress.

## Usage

Run the forked CLI (build instructions below), then in a session:

```
/forget              # forget the last user turn (soft)
/forget 3            # forget the last 3 user turns (soft)
/forget 2 --hard     # forget the last 2 user turns and rewrite the session file
```

Status output after a soft forget:

```
Removed 4 message(s) (1 user, 2 assistant, 1 toolResult, ~603 tokens) from the model context (removed turns stay in the file; /tree can navigate back).
```

After a hard forget the scope reads "the session file and the model context". The role breakdown in parentheses shows exactly which entries were removed.

Errors:

- More turns requested than exist: `Session has only 2 user message(s); cannot forget 3.`
- Bad count: `Count must be a positive integer`
- While busy: `Wait for the current response to finish before using /forget.`

## What is (and is not) removed

**Removed** from the model context (and, in hard mode, from the file): the Nth-last user message and everything after it on the active path — assistant messages, thinking blocks, tool calls, tool results.

**Not touched:**

- **Out-of-band state.** Files written, memory files, `AGENTS.md`, tool state, or anything outside the session file. `/forget` erases the conversation, not its side effects.
- **Earlier turns** on the retained path, including any compaction entries (hard truncation remaps their references correctly — covered by unit tests).
- **Other sessions.** The operation is scoped to the current session file.
- **Provider-side history.** Requests already sent to the model provider are not retracted. The guarantee is about what the model sees in *subsequent* requests in this session.

## Hard-mode semantics (read before using)

- The session file is rewritten to `header + retained path`.
- **Abandoned branches are also deleted** — if you previously branched with `/tree`, those alternative paths are gone too.
- **No backup is written, by design.** There is no undo. Treat `--hard` as `rm`.
- Soft mode is the default precisely so mistakes are recoverable: everything stays in the file and `/tree` can navigate back to the removed turns.

## Limitations

- **The no-awareness guarantee is conditional on how you use `/tree` afterwards.** A soft `/forget` leaves the removed turns as an abandoned branch. If you later navigate away from that branch with `/tree` and *accept a branch summary*, a summary of the removed content is attached to the context and the model learns about it again. Choose "no summary" (or never accept a summary of the removed branch) to preserve the guarantee.
- Token counts are approximate (`~N tokens` uses a chars/4 heuristic, not the provider tokenizer).
- `N` counts user messages on the active path (the current branch). Injected `custom_message` entries are not counted as user turns.
- The harness does not own the provider's KV cache. The shortened message list is what gets sent, and correctness does not depend on cache invalidation — but no explicit cache flush is issued either.
- The command is wired into the interactive TUI. The underlying `AgentSession.forgetMessages(count, { hard })` API is also available programmatically (extensions, RPC, SDK).
- Fork maintenance: upstream is active. The delta is small and self-contained, but expect occasional rebases.

## Building and running

```bash
git clone https://github.com/robert896r1/pi
cd pi
npm install
cd packages/ai && npm run hydrate-model-data && cd ..
# build in dependency order:
for p in tui telemetry ai chord agent protocol client coding-agent; do
  (cd packages/$p && npm run build)
done
node packages/coding-agent/dist/bundle/cli.js
```

Requires Node >= 22.19. (`bun` is only needed for the standalone binary, not for this.)

## Testing

```bash
cd packages/coding-agent
npx vitest run test/agent-session-forget.test.ts test/session-manager/truncate.test.ts
```

15 unit tests (no API key required) cover: soft/hard behavior, file rewrite with no backup, forgetting all turns, continuation after a soft forget, user-boundary cuts, turns containing tool calls, label preservation, compaction reference remapping, unflushed session files, in-memory sessions, and error cases.

End-to-end verification (local Qwen 27B, hermetic session): the model recalled a unique secret before `/forget --hard`, the rewritten file contained no trace of it, and the model answered "I don't know" when asked for the secret afterwards.

## Files changed (vs upstream)

| File | Change |
|------|--------|
| `packages/coding-agent/src/core/session-manager.ts` | `truncateTo(entryId)` — moves the leaf and rewrites the file to header + retained path |
| `packages/coding-agent/src/core/agent-session.ts` | `forgetMessages(count, { hard })` + `ForgetResult` (incl. `removedByRole` breakdown) |
| `packages/coding-agent/src/core/slash-commands.ts` | Registers `/forget [N] [--hard]` |
| `packages/coding-agent/src/modes/interactive/interactive-mode.ts` | Dispatch, argument parsing, hard-mode confirmation, status line |
| `packages/coding-agent/test/session-manager/truncate.test.ts` | New — 8 tests |
| `packages/coding-agent/test/agent-session-forget.test.ts` | New — 7 tests |
| `packages/coding-agent/README.md`, `docs/usage.md`, `docs/sessions.md` | Command-table rows + `/forget` section in `sessions.md` |