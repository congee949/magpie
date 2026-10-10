# Streamed reasoning speed verification

Base: `0fb4ff17ba089983fa4976162476a15dbd5390be`, 2026-10-10.
This revises the proposal in [#1463](https://github.com/yetone/magpie/issues/1463#issuecomment-6094507589), not the unrelated thread-description PR #1198.

## What changes

A no-answer turn can contribute output speed only with paced visible reasoning
and usable content flow. Answer text still has its own token count and window.
Hidden, encrypted or redacted thinking is excluded even without a separate
reasoning-token count. The ledger, Routing, Usage, compressed row cache and
Codex own-login backend retain the evidence. The [subsystem reference](../subsystems/gui-shell.md#streamed-output-speed)
owns the fields and limits.

Responses/Gemini summaries are conservatively excluded from no-answer speed.
Their visible summary text does not account for the full reported reasoning count. Native
Responses tools without reasoning keep their previous timing. A known summary
upstream is marked only when it actually sends reasoning, not for every reply.

Receive-side pacing is an observed output-rate estimate. It cannot identify a
relay that releases buffered content evenly without an upstream timing guarantee.
The existing held-flow cut rejects concentrated delayed releases, and tests cover
that case; evenly paced buffered release remains a measurement limitation.

## Real upstream samples

The [sanitized sample index](reasoning-speed-live-20261010.json) preserves event
types, counts, channel receive-time ranges, usage counts, status and decode results. It
omits account names, hosts, IDs, credentials and response text. Raw copies stay
outside the repository. All requests use synthetic arithmetic and a `check_value`
tool: calculate `17*23 + 29*31`, then answer, or call the tool. No tool was executed.

- WorkBuddy's moved community plugin, DeepSeek v4.1 flash: the tool turn returned
  76 output / 37 reasoning tokens, no answer text, first reasoning at 6,787 ms,
  end at 7,518 ms, flow 187 ms. The 76/731 ms estimate is about 104 tok/s. The
  answer turn timed only 24 answer tokens over 465 ms. Later sampling was filtered
  as held content; a named tool-choice object was refused by WorkBuddy (400).
- Codex gpt-6.1-sol: one tool request returned 529 overloaded; a later tool request
  returned 44 output / 23 reasoning tokens and no text, with no speed. Answers
  used their answer window, such as 41 tokens / 1,586 ms. A separate no-reasoning
  tool turn returned 19 tokens / 322 ms and keeps its ordinary speed.
- Claude Haiku 5.5 through the genuine Claude Code bridge: the first samples'
  visible deltas arrived in bursts (1–2 ms content flow), so no speed was shown.
  The tool request also wrote text, so it is not evidence of an omitted-thinking
  tool-only turn. A later sample hit the existing reply-loop guard; HTTP 200 with
  a stream error is recorded as failed, not as a successful speed check. The
  answer-only sample without reasoning retained its ordinary speed.
- Early Codex/Claude 404s came from missing copied auth/catalog files in the
  initial isolated HOME. The corrected run used copied files, never live writes.

These cases verify visible Chat reasoning, hidden Responses, normal text,
no-reasoning tool turns and burst handling. A genuine Claude omitted-thinking
no-text tool-only sample was not obtained; synthetic sealed/mixed regressions
cover the exclusion rule, and this missing live case stays explicit.

## Regressions

`TestVisibleReasoningSpeed` covers the 100 ms / 10,000 tok/s / quarter-flow
boundaries, unknown flow, invalid token counts, hidden reasoning and cache
round trips. Gateway tests cover three client protocols, hidden translations,
multiple visible blocks, mixed redacted blocks, encrypted reasoning with no
count, Gemini summaries, native Codex and slowly released concentrated bursts.
`TestLiveVisibleReasoningFixture` replays the actual DeepSeek reasoning text and
relative receive times from `testdata/reasoning-speed-visible.json`.

The GUI regression covers Routing rows, sessions, Usage rows and the period KPI
on Chromium/WebKit in en, zh, zh-TW, ja and de. All three formulas stay aligned.

```sh
go test -race -tags nogui ./internal/gateway ./internal/usage -run 'Test.*Reasoning.*|TestVisibleReasoningSpeed' -count=20
NODE_PATH=/path/to/node_modules node --test --test-concurrency=1 internal/gui/tests/reasoning-speed.test.cjs internal/gui/tests/usage-speed.test.cjs internal/gui/tests/routing-burst-speed.test.cjs
env -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY -u NO_PROXY -u http_proxy -u https_proxy -u all_proxy -u no_proxy -u NODE_USE_ENV_PROXY go test -tags nogui -p 4 ./...
```

The full-suite invocation with the host's proxy variables and PATH Bun 1.3.14
failed `TestBunThroughBridge`, `TestPluginHTTP2TakesTheProxy` and `TestOffline`.
Clearing HTTP(S)/ALL/NO_PROXY in both cases, and selecting the application's
existing Bun 1.4.2 on PATH, passes those three focused checks. This changes only
the test process environment, not the user's proxy settings or installed Bun.
The full suite is repeated under that environment; its final outcome is recorded
in the issue update. The earlier failed logs remain local.

Test status is recorded at the submitted head in the issue update. Linux and
Windows vet/build are cross-platform compilation checks, not runtime checks.
The live gateway and installed application were not replaced. No release is
part of this branch.
