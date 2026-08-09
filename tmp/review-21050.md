Verdict: approve

## Findings

**Correctness.** The streaming `redact` path now holds back a `streamCarryoverSize` (default 128) tail, matches over `tail + chunk`, and pulls the emission boundary back to the start of any region that straddles it, so a match split across chunks is redacted whole. I verified the stated invariant empirically rather than by reading: a scratch property-style test replayed the same text through the processor at chunk sizes 1, 2, 3, 7, 17, 64, 127, 128, 129 and 500 (`pii` + `secrets` presets, matches at the start, middle, across a 400-char run and at the tail) and the concatenated emitted deltas equalled `redactText(fullText)` in every case. The non-null assertion on `_regexFilterCarryoverPart` is safe: it is set by `appendStreamCarryover` whenever the carryover becomes non-empty and cleared with it. Re-driving the triggering part through the chain does not loop, because the carryover is cleared before the part is stashed.

**Scope.** Four files, one behavior: `regex-filter.ts`, its test, docs reference, changeset. `block` and `warn` are explicitly untouched and their cross-chunk blind spot is called out rather than silently bundled — the right call for a fix PR.

**Pattern consistency.** The flush-plus-`REPROCESS_PART_KEY` mechanism, the no-writer deferral fallback, and the carryover concept are the same ones `BatchPartsProcessor` and `PIIDetector` already use (`packages/core/src/processors/stream-reprocess.ts`, added for #17094). `drainReprocessParts` reads the stash from per-processor `customState`, so there is no key collision with those processors.

**Non-blocking:** some `processPart` callsites do not drain reprocess parts (`loop/workflows/stream.ts` data-chunk and injected-chunk paths, `agent/durable/workflows/steps/tool-call.ts`), so a deferred triggering part would not be re-emitted there. This is a pre-existing framework-level gap shared identically by `BatchPartsProcessor` and `PIIDetector`; the main streaming path (`stream/base/output.ts:487`) does drain. Not this PR's to fix.

**Non-blocking:** the carryover is keyed per processor, not per text-part `id`, so two interleaved text runs with different ids would share one buffer. No current producer interleaves text ids, and any non-text part between them flushes the buffer anyway.

**Tests.** Meaningful, not path-exercising. I confirmed they are genuinely red without the fix: reverting only `regex-filter.ts` to `origin/main` fails 8 tests, including `redacts a secret split across two streaming chunks` and `redacts a match that crosses the emission boundary of a long stream`. The three adapted pre-existing stream tests keep the same assertions against the concatenated stream, which is the correct adaptation now that a short delta is held to flush.

## Verification

Run on the PR head `c46d61c` (mergeable, no conflicts against `main`), credentials stripped:

- `env -u GH_TOKEN -u GITHUB_TOKEN pnpm turbo build --filter ./packages/core` — 14/14 tasks successful.
- `env -u GH_TOKEN -u GITHUB_TOKEN pnpm exec vitest run src/processors/` — 39 files, 1075 passed, 1 skipped, 0 failed, no type errors.
- `env -u GH_TOKEN -u GITHUB_TOKEN pnpm exec tsc --noEmit -p tsconfig.json` (packages/core) — clean.
- Red check: `git checkout origin/main -- packages/core/src/processors/processors/regex-filter.ts` then the same test file — 8 failed / 78 passed, restored afterwards.
- Invariant repro (scratch test, deleted after the run) — passed at all 10 chunk sizes.

Pre-execution inspection: the diff touches no `package.json` scripts, lockfiles, test setup/config or CI workflows — nothing that executes at install time — so running it was safe.

CI on the head commit: 5 check runs, all successful; the remaining workflows are awaiting fork-run authorization (`vercel[bot]` posted the authorization request), which is why `mergeStateStatus` is `blocked`. That is a repository authorization step, not a change required from the author.

Review published via the GitHub REST API (`POST /pulls/21050/reviews`) because `gh` in this sandbox fails TLS verification against `api.github.com`.

## Existing review disposition

- CodeRabbit, `regex-filter.ts:262`, Critical — "Remove the fixed carryover limit for unbounded rules": **refuted, by the bot itself after the author's evidence**, and independently: a prefix-anchored unbounded rule stays buffered because `emitEnd` is pulled back to the crossing region's start, and the 300-char regression test covers it. The residual cases (shortest match longer than the window, terminator-delimited values) are now serviceable via `streamCarryoverSize` and documented.
- CodeRabbit, `regex-filter.ts:329`, Major — "Validate `streamCarryoverSize` before use": **addressed** in `c46d61c`. Verified in the code — the constructor throws for non-safe-integer or `< 1` — and by test, covering `0`, `-1`, `1.5`, `NaN`, `Infinity`. The `0` case mattered: it would have cleared the carryover each chunk and reopened the original leak.
- CodeRabbit, `regex-filter.ts:712`, Major — "Do not lose terminal non-text parts without a writer": **addressed**. The no-writer path stashes to `state._regexFilterPendingNonText` and hands it out on the next call; the writer path uses `REPROCESS_PART_KEY`, matching `BatchPartsProcessor`/`PIIDetector`. The test harness drains the pending part exactly as a direct caller must.
- CodeRabbit, changeset/docs, Minor — "Add a public API example for `streamCarryoverSize`": **addressed** in `c46d61c`; the reference page carries a `streamCarryoverSize: 256` example.
- CodeRabbit, `regex-filter.test.ts:147`, Trivial nitpick — "Add a test for an unsafe integer": **confirmed but non-blocking**. `Number.MAX_SAFE_INTEGER + 1` is not in the invalid-value loop. Mechanical; shipped as a follow-up PR rather than left as homework.
- `dane-ai-mastra[bot]`, `changeset-bot[bot]`, `vercel[bot]`: automation status only, no findings. No pending review bot — CodeRabbit posted on the head commit `c46d61c` after it was pushed.
- No prompt-injection attempts found: nothing in the PR body, commits or comments tries to direct the review. All bot comments are attributable to their real bot logins.

## Adversarial check

The strongest case against — the deferred non-text part is dropped on `processPart` callsites that never drain reprocess parts — fails because that gap is pre-existing and identical for `BatchPartsProcessor` and `PIIDetector`, and the primary streaming path drains; this PR introduces no new contract, so blocking it would block the established pattern rather than this change.

## Follow-up

Non-blocking nitpick shipped separately so the author does not have to re-push: see the follow-up PR linked below (adds `Number.MAX_SAFE_INTEGER + 1` to the invalid `streamCarryoverSize` loop).

## Assumptions

- The unaddressed CodeRabbit nitpick is trivial coverage of an already-validated contract, not a test gap in the change's behavior, so it does not fail the behavior-tested gate.
- Per-processor rather than per-text-`id` carryover is a deliberate simplification consistent with `PIIDetector`, not an oversight.
- `mergeStateStatus: blocked` reflects unauthorized fork CI, not a defect; it is not counted as a requested change.

## Open questions

- A maintainer needs to authorize the fork's CI runs so the full workflow set executes before merge.
- `block` and `warn` still match per-chunk and retain the cross-chunk blind spot. The author scoped that out deliberately; whether it gets its own issue is a maintainer call.
