# Sync to pi 1.3.0: three more retryable provider wordings, devDeps bumped, all internal-surface seams re-verified

The host pi moved 1.1.0 → 1.3.0 (mise, 2026-10-15), so this ADR records the
re-verification obligation from ADR 0009 and the sync pattern from ADRs
0010/0011/0013 for the 1.3.0 series. pi 1.2.0's breaking changes are all
outside this extension's surface (`BeforeAgentStartEvent` split into a
`messageType` union, session-busy semantics for `sendMessage`/`isIdle`,
`pi mcp list --json` shape, theme `$schema` validation — the fork uses none of
those symbols; `config.json` defines no theme); pi 1.3.0 has no breaking
changes. The only behavior-relevant drift is in pi-ai 1.3.0's
`dist/utils/retry.js`, where three retryable provider patterns were added
under Fixed (semantic changes hiding in Fixed, per the sync checklist). The
sync diff is the smallest possible: three retryable patterns transcribed
verbatim, devDeps bumped, line-number stamps refreshed against the 1.3.0 dist.

## Decision

- **`retry.js` mirror gains three provider wordings verbatim** from pi-ai 1.3.0
  `dist/utils/retry.js`, verified with a full (untruncated) byte diff of the
  1.1.0 and 1.3.0 files (1.1.0 re-obtained via `npm pack @earendil-works/pi-ai@1.1.0`):
  - `"service.?unavailable"` → `"service.?(?:is.?)?(?:temporarily.?)?unavailable"` —
    now also matches "The service is temporarily unavailable" (#10965).
  - `"provider.?returned.?error"` → `"provider.?returned.?(?:an.?)?error"` —
    now also matches "The upstream provider returned an error" (#10842).
  - added `"unable to forward upstream"` — gateway failure relaying the upstream
    response stream, sent as an OpenAI Responses `response.failed` event
    without an error code (#10920).
  Both regex relaxations are pure supersets of the previous match sets (new
  patterns match everything the old ones did), so there is no regression on
  previously classified texts. **Behavior-affecting**: with a provider that
  reports these wordings as an error message, the fork's turn loop now retries,
  matching the main session (ADR 0013's parity rationale applies unchanged).
- **Decision order re-verified**: the composition point moved from pi-agent-core
  1.1.0's `publishResponse` to pi-coding-agent 1.3.0's `agent-session`
  (`dist/core/agent-session.js` L3013-3015): overflow branch first, then the
  retryable branch — unchanged semantics; the fork mirrors pi-ai's
  `retry.js`/`overflow.js` tables directly.
- **devDependencies bumped** `^1.1.0` → `^1.3.0` for all four
  `@earendil-works/pi-*` packages; `bun run typecheck` is green against 1.3.0
  and the full suite passes (412 tests). Import surface re-verified: top-level
  entries, `pi-ai/compat`, and the deep import
  `pi-tui/dist/word-navigation.js` all resolve against 1.3.0.
- **All internal-surface seams re-verified against the 1.3.0 dist** (the ADR
  0009 upgrade obligation; the new numbers live in the source comments):
  - pi-tui `dist/tui.js`: full diff is exactly the fake-cursor machinery
    (constants + `renderFakeCursor` after L69, `resolveFakeCursors` wrap at
    L978, new `resolveFakeCursors` method after L1056) — additive, inert for
    the fork (the fork emits no fake-cursor markers). `resolveOverlayLayout`
    L827-915 → **L836-924** (L834 is its doc-comment line, not the method);
    `parseSizeValue` L72 → **L81**; rendered-height resolve call L975 →
    **L984**; maxHeight composition clamp L855 → **L864**.
  - pi-tui `dist/tui-alt-screen.js`: full diff is two fake-cursor wraps plus
    removal of the `shouldDeferViewportInputToOverlay()` wheel branch (#10769,
    a pi-side transcript-scroll fix) — not mirrored by the fork (the fork
    translates `handleMouse` coordinates; it does not transcribe alt-screen
    input routing).
  - pi-tui `dist/word-navigation.js`/`.d.ts`, `dist/native-platform.js`,
    pi-ai `dist/utils/overflow.js`, `dist/compat.js`: byte-identical.
  - pi-coding-agent `dist/core/sdk.js`: byte-identical — the
    `injectProviderRetry` `??`-fallback injection point (`buildRequestOptions`
    L193-211, `streamFn` L262) is unchanged.
  - pi-coding-agent `dist/core/extensions/runner.d.ts`:
    `getAllRegisteredTools` L132 → **L134** (shift comes from the
    `BeforeAgentStartTrigger` import-line addition and the
    `emitBeforeAgentStart(trigger, …)` signature change; the fork uses neither).
- **Version comments refreshed** across `srcs/` and `test/` (1.1.0 → 1.3.0)
  and READMEs (both languages) pin 1.3.0. Line-number claim corrected during
  the sync: the first pass wrote `resolveOverlayLayout` as L834-924 (the
  doc-comment line); the method starts at **L836** — both source comments were
  corrected to L836-924.

## Considered options

- **Adopt only the pure addition (`unable to forward upstream`), skip the two
  relaxations** (rejected): partial adoption creates a fork-specific table
  pi does not have, breaking the "照抄语义，不发明私有协议" contract; the two
  relaxations are pure supersets (no regression) and are exactly the wordings
  pi deliberately chose to retry (#10965/#10842).
- **Skip the retryable-table additions** (rejected, same rationale as ADR
  0013): the fork's retry decision would silently diverge from the main
  session — the mirror's whole purpose is parity.
- **Give the fork its own retry semantics** (rejected): no expressed demand;
  a new contract and configuration surface that would conflict with the
  single-source-of-truth `settings.retry` block.
- **Rewrite line-number claims in older ADRs** (rejected): per the convention
  recorded in ADRs 0008/0009/0010, older ADRs are historical records of what
  was verified at decision time; the live source comments carry the fresh
  numbers.

## Consequences

- `test/retry.test.ts` gained the three wordings in the retryable list fixture
  at the `classifyRetryable` seam ("The service is temporarily unavailable",
  "The upstream provider returned an error", "Unable to forward upstream
  stream", covering the exact issue wordings); the full suite stays green
  (412 pass) and `bun run typecheck` is green against 1.3.0.
- The behavior-affecting item (these wordings now retried by the fork) is
  code-inferred; runtime confirmation [NEEDS MANUAL VERIFICATION].
- Released as v1.7.2 (patch, per the sync-release convention from 1.6.2/1.7.1),
  carrying the un-released ADR 0014 (direct-flow publish) along.
- ADRs 0001–0014 remain historical records, untouched.
