# September request-id stream join — 2026-09-26

## Baseline and symptom

- Baseline: official `main` at `750fad9378a0cf9e37791916b11f7ed9add645dd`.
- The 2.1.14 release payload still used Fiber v13 and could not read the current ChatGPT shell at all. Current `main` uses Fiber v21 and restored shell/turn observation, but exact MCP request attribution still arrived too late.
- On the affected live page, the companion knew the exact chat and recorded tool calls, while `Current request` remained unavailable during the MCP identity window. Core calls then logged `no page evidence`, waited roughly fifteen seconds, and were filed under Unattributed activity before later repair.
- This broke chat-scoped agent ownership in the same way reported in #407: a worker could be opened before its prime request had an exact conversation, and a worker finish could likewise race attribution.

## Root cause and incorporated upstream work

Incorporates the request-origin fix from [PR #414](https://github.com/totec448-spec/chat-on-steroids/pull/414), authored by [@Maximapple](https://github.com/Maximapple) at commit `c9ecc3eb14f4b8094a06f61f65511da8c0fadc10`. Live stream measurements cited by that PR were provided by [@moderntanri](https://github.com/moderntanri) in #393.

Three independent parser assumptions had become false:

1. `/backend-api/f/conversation` now carries the current request id at `input_message.metadata.request_id` as well as the older supported locations.
2. The response can split the exact join across consecutive SSE events: one event names `conversation_id`, while a later `input_message` event in the same HTTP response names the request id but no conversation.
3. Stream-origin evidence is produced before ChatGPT has mounted a provider message, so `/correlations` legitimately receives `messageId: null`; the parser previously discarded those bare request-id rows before the route could use them.

The MAIN-world usage observer now retains only the single root response-local conversation id needed to join later events in that same response. During final review, the upstream implementation was hardened further: a nested `conversation_id` cannot seed the cache, and an ambiguous/multi-conversation frame clears any cached owner before later request-only events. Bare `/correlations` rows are deduplicated by request id when no message id exists.

No sole-active-chat, sole-family, `run_id`, focus, or timing fallback was added. Exact request/conversation evidence remains the ownership authority.

## Fresh-worker first-turn binding race

Live validation after the stream-origin repair exposed a second, narrower race. A brand-new worker could open, receive its task, and run tools, yet its first `agents` report returned `AGENTS_BUSY: no agent family belongs to this conversation`. The same worker's second turn succeeded after its tab had promoted to the canonical `/c/<id>` route.

The worker slot is intentionally created as `invited` with no conversation. After native Send, `extension/content.js` waits for both the exact browser route and the authored user-row receipt before ACKing the bootstrap; only that authoritative ACK lets `bridge.ts` call `bindConversation()` / `activateWorker()`. ChatGPT can begin the model turn and issue an exactly attributed MCP request before those browser facts mount. At that point caller identity is exact, but broker membership is legitimately still pending.

An intermediate attempt added a bounded membership wait inside `callerNow()`. Review found that its trigger used global `pendingWorkerSpawns()`, creating cross-family latency coupling even though it granted no authority. That approach was removed.

The final repair binds at the stronger existing authority boundary instead. If a fresh worker's stream request-origin arrives after the concrete browser route but before native Send has returned far enough to install the redeemed `agentCommandId`, `content.js` retains that origin instead of consuming it with worker proof missing. Installing the exact command id immediately re-flushes the retained origin. `background.js` forwards the worker label and command id only after `currentConversationDocument()` proves that exact document is still on the named conversation. `/correlations` then reuses the same `(agent, random leased command id)` validation already used by `/events` lost-ACK recovery before calling `bindConversation()`. This lets exact request attribution and exact worker membership converge on the same early correlation path without waiting on or observing unrelated families.

Regression coverage proves that correlation without the exact command id records request ownership but does not bind a worker, while the same concrete conversation plus the exact redeemed command activates/binds the worker. A shell ordering regression injects the request origin during the worker's native Send and proves there is no proofless correlation; once `agentCommandId` is installed, exactly one correlation is emitted with both `agent` and the redeemed command id.

## Review of upstream PRs opened after the September break

All official pull requests from #411 through #425 were reviewed against this failure before finalizing the local repair. #414 is the request-origin parser fix incorporated above. #411 fixes a different case where an unattributed/null-home spawn may never open a worker tab at all; our failing worker tabs did open and execute. #417 explicitly documents that an explicit worker finish may still hit `AGENTS_BUSY` when family attribution is unavailable and deliberately leaves that ownership boundary unchanged. #418 fixes the new localized/slot-based Send control; #420 adjusts long silence handling after attribution is repaired; #422, #423 and #425 address stale generating state, shell turn identity and recovery text residue. None of those PRs binds the fresh worker before its first membership-sensitive `agents` call. #412, #413, #416, #421 and #424 concern stall reporting, model display, tunnel diagnostics or recovery logging and are not on this binding path.

## Temporary diagnostic and cleanup

A local Fiber v22 structural probe was used only to confirm the live shell eventually exposed the same exact `metadata.request_id`. It reported bounded field names/types plus opaque id-shaped values and never participated in attribution. After PR #414 restored immediate stream attribution and the end-to-end worker flow passed, that diagnostic code was removed; production Fiber/recorder versions remain at the baseline values.

## Validation

- Live browser, freshly loaded ChatGPT document after installing the patched Windows package:
  - first `agents status` resolved directly to the prime family with no Unattributed identity notice;
  - prime-to-worker wake/message was accepted normally;
  - a reused worker resumed in its existing chat, completed its read-only check, queued its report to prime, and finished without `AGENTS_BUSY` or identity failure;
  - a newly spawned worker on the earlier build reproduced the first-turn membership race above; its second turn succeeded after canonical route/broker promotion. Inspection of the live timestamps showed request attribution at 09:36:18, first `AGENTS_BUSY` at 09:36:38, and authoritative worker binding only at 09:36:54. That experiment helped isolate the missing exact-command proof on the early correlation path.
  - final `final3` acceptance used a brand-new `worker-4` whose first requested tool action was an `agents` report to prime. The app bound `worker-4` to conversation `6ab79acb-4618-83eb-bacf-5552a46910e5` at 10:13:35.094, attributed the same worker request at 10:13:35.100, accepted the first `agents` call at 10:13:42.425, and accepted `finish` at 10:13:51.459. The worker reported `FINAL3_FIRST_TURN_FINISH_OK`, with no `AGENTS_BUSY`, Unattributed, or identity error. This proves the exact-command `/correlations` recovery closes the first-turn family-binding race before the worker's first membership-sensitive call.
  - the earlier `final2` package was also found to contain stale `out/main/index.js` because electron-builder had been run without rebuilding after an intermediate Core source change. `final3` was rebuilt first and its installed mirrored extension was directly checked for the current `agentCommandId` correlation path before the acceptance test.
- `test/usage-observer.test.ts`: 34/34 pass, including split-event request-id join, contradictory-conversation refusal, nested-id refusal and multi-conversation cache retirement.
- `test/bridge.test.ts`: 530/530 pass, including bare `messageId: null` correlation evidence, request-id deduplication, and exact-command fresh-worker binding from `/correlations`.
- `test/agents.test.ts`: 164/164 pass after removing the discarded global membership-wait regression.
- `test/shell-compat.test.ts`: 95/95 pass, including an early-origin ordering regression that forbids proofless worker correlation and then requires the exact redeemed command id.
- Existing shell/popup compatibility tests remained green during the diagnostic phase: 100/100 pass.
- `npm run typecheck`: pass.
- `npm run build`: pass.
- An earlier diagnostic `npm run dist:x64` completed successfully. Later packaging while Chat On Steroids itself was still running hit `EBUSY` on the default `release/win-unpacked/Chat On Steroids.exe` staging file, so final packages use the same electron-builder configuration and `COS_PACKAGE_ARCH=x64` with a temporary alternate output directory. The final rebuilt artifact for live acceptance is `release/Chat-On-Steroids-Setup-x64-final3.exe`, 164,361,165 bytes, SHA-256 `BA55F542652F9D2412FD68F7FFAD210D178D8EDD0FD7917A7767CD996DBF579D`; its temporary `release-final3` staging directory was removed after the hash-verified copy.
- `git diff --check`: pass.
- Final `npm run verify` after removing the temporary diagnostic and adding the fresh-worker early-origin ordering regression:
  - primary CI suite: 222 files passed / 5 skipped, 5,829 tests passed / 46 skipped;
  - `test/mcp-shutdown.test.ts`: 6/6 pass;
  - `test/computer.test.ts`: 17/20 pass; the same three pre-existing Windows screenshot tests fail at `STALE_FRAME: capture and DWM bounds disagree`, independently reproduced on the clean `750fad...` baseline before this request-attribution change.

The unrelated `THIRD-PARTY-NOTICES.txt` change produced by the earlier dependency/install workflow was restored before finalizing this patch.

## Commit attribution

If this adapted integration is committed rather than merging PR #414 directly, preserve the original author with:

`Co-authored-by: Maxim <5410641+Maximapple@users.noreply.github.com>`
