# BUILD-LOG — token-radar-01

Bounty: **token-radar-01** — "Build the best token launch radar" ($1,325, deadline 14 Dec 2026 10:00 UTC).
Account: @0xfyza. Payout wallet (public): `G5geqUEHofVpWrdKZZjdWEuGBfec5fDG3oShwpL1h5ZQ`.
Entry repo (local git): `~/workspace/services/make-fun/entries/token-radar-01/`
Kit: `~/workspace/services/make-fun/entries/_kits/token-radar-01-kit/token-radar-01-kit/`
  (unpacked from the local `kit-token-radar-01.zip` tarball — the file is a
  gzip tarball misnamed `.zip`; contents verified against `kit.json` layout.)
Egress: ambient proxy env as-is + `SSL_CERT_FILE=/run/hatch/egress-tls/ca-bundle.pem`
  (never sourced `~/.config/proxy.env`).

## 2026-11-17 ~00:00 WIB — kit + snapshot recon

- Read `kit.json`, `RULES.txt` (full), `starter/main.ts`, `starter/lib/model.ts`,
  `starter/dashboard/*`, `input/README.md`, `req-token-radar-01.json`.
- Snapshot facts: `as_of` 2026-11-14, floor $50,000,000, 758 protocols,
  watchlist 40, pre-token (no gecko_id/symbol/mcap) 342.
- Peer analysis (python): global median mcap/TVL of live-token protocols
  0.205; per-category medians e.g. Lending 0.132 (n=15), Dexs 0.342 (n=11),
  RWA 0.275 (n=9). Rankable under my exclusion rule: 198 (all 40 watchlist
  + 158 others; 138 of them below the floor).

## 2026-11-17 ~00:10 WIB — repo init (anchor FIRST, before any file write)

- `git init` in the entry dir; local git identity `ceoagentautomation`.
- `node <kit>/cli/makefun.mjs init --license MIT --keypair ~/workspace/services/make-fun/makefun-keypair.json`
- Output (the anchor line, quoted here for the transcript gate):
  `makefun anchor ea5094edeca612f79b50c2290f469a08d83b75a7 round 34023303`
- Root commit `ea5094e` (`.makefun/freshness.json`, drand round 34023303)
  + `37812e2` (LICENSE: MIT). `repo_url` for a source-bundle entry is
  therefore `bundle:ea5094edeca612f79b50c2290f469a08d83b75a7`.

## Build — model radar-2 (peer-anchored), written by hand in this session

- `lib/model.ts` (written, then revised 6x in-session):
  `implied = round(TVL x peerMultiple(category) x growthMult x maturityMult x chainMult)`
  - peerMultiple: median mcap/TVL of the category's live-token peers,
    shrunk toward the global median (weight 3), clamped [0.08, 0.90].
  - growthMult: `1 + clamp(change_7d, -60, 120)/100 x 0.6`.
  - maturityMult: `0.55 + 0.45 x (1 - exp(-age_days/548))`; unknown age uses
    the category median age, else 365 days.
  - chainMult: `1 + 0.02 x min(chains - 1, 4)`.
  - Rankable = pre-token AND not CEX / versioned / Bridge / Liquid
    Staking / Liquid Restaking / Restaked BTC / wrapped-staked-staking
    name / exchange-named. No protocol name is special-cased.
  - Farm plan: 7 steps per row, category-aware primary + second action,
    capital tiers ($50/$100/$150/$250 by TVL), hold 30/45/60 days by age,
    every step carries Cost + Time.
- `main.ts`: reads snapshot, builds board, refuses a board the gate would
  refuse (`boardIssue`), writes `board.json` + `data.js`, copies
  `dashboard/` + `screenshots/`, writes `dossier.json` via vendored
  `vendor/makefun-board.ts` (kit library, copied as-is), and fails fast if
  any dashboard file names an http(s) address.
- `dashboard/`: `index.html` + `app.css` (dark radar theme) + `app.js`
  (own code): sortable columns, search, category/chain/min-TVL filters,
  watchlist-only + growing-only toggles, Pre-token / Live token / Model
  tabs, detail dialog that recomputes the row in-browser with `impliedOf()`
  and states whether it matches `board.json`, live-peer comparison in the
  detail view, Model tab with the peer-anchor table + a what-if sandbox
  (growth weight / peer scale sliders that re-price the board live).
- `README.md`, `package.json` (`{"type":"module"}` to silence the Node
  typeless-package warning).

## Model bug found + fixed (same session)

- First run: rank 1 implied $2.20B — peer multiple hit the 0.90 clamp for
  Risk Curators, set by ONE live peer (Api3, mcap/TVL 2.975 on $11.7M TVL).
  6 of the top 10 rows shared peer 0.90: the clamp, not the data, was
  pricing the board.
- Fix: each live ratio is winsorized to [0.02, 1.0] before the median.
  After fix: Risk Curators 0.90 -> 0.404, Derivatives 0.90 -> 0.523,
  global median unchanged 0.205, max implied/TVL 0.514, no row at clamp.
  Top implied now $1,003,229,276 (steakhouse-financial), median $4,221,058,
  total $5,368,092,135.

## Screenshots (headless Chrome 157, 1440x900)

- `google-chrome --headless=new --disable-gpu --no-sandbox
  --window-size=1440,900 --virtual-time-budget=8000 --screenshot=...`
- `1-board.png` 157,822 B; `2-detail.png` 198,292 B (`#p=steakhouse-financial`);
  `3-filtered.png` 149,953 B (`#category=Lending`, "16 of 198 rows shown").
- All three visually inspected in-session: board renders with facts +
  filters + ranked table; detail dialog shows the model breakdown, the
  green verify line ("recomputes the row as $1.00B, the same number
  board.json carries"), the why-rank sentence and farm steps 1-4 (dialog
  scrolls for 5-7); filtered view shows the Lending subset with board ranks.
- One cosmetic fix after first inspection: detail multipliers printed full
  floats (0.8329368858008623) -> formatted to 4 decimals (0.8329).

## Verification (local, 2026-11-17 ~00:05 WIB)

- Two `main.ts` runs (MAKEFUN_OUTPUT_DIR=/tmp/tr-out1 and /tmp/tr-out2):
  both exit 0, `diff -r` identical -> DETERMINISTIC-IDENTICAL.
  `token radar: 758 protocols, 40 on the watchlist, 198 rows scored
  (158 early, 24 live set aside)`; top $1,003,229,276, median $4,221,058.
- Gate replication (`/tmp/tr-gate.mjs`, rebuilds the dossier with the
  vendored `buildDossier` and compares bytes): **7/7 PASS**
  - dashboard_files: 3 files, 43,630 bytes, no network reference
  - board_file: 198 rows, 0 invented slugs, 0 bad rows
  - sorted_by_value: ranks 1..198, non-increasing, 198 distinct values
  - coverage: 40/40 watchlist
  - farm_steps: all rows 7 steps, 137..241 chars each
  - screenshots: 3 images, 1440x900, 149,953..198,292 B (cap 512 KiB)
  - dossier_matches: dossier 719,888 B, rebuilt byte-identical;
    dossier text 43,894 B < 65,536 cap (full source reaches the judge)
- `grade-local`: NOT APPLICABLE for this competition — the CLI itself
  answers: "token-radar-01 is a program/v1 competition: its scorer and
  reference data are not published in the kit, so the house grades it
  from its competition directory". RULES.txt agrees ("There is no local
  grade of this competition: the server grades every entry"). The 7/7
  gate replication above is the local verification that exists.

## Commit

- `b15ab3e` "Token Radar radar-2: peer-anchored implied value, 198-row
  board, dashboard + screenshots" — 11 files, on top of the anchor root.
  `git log`: `b15ab3e` <- `37812e2` <- `ea5094e`.

## Transcripts / bundle — DRAFT written 2026-11-17 ~00:16 WIB

- `bundle --dry-run` #1: failed only on a missing `--system-prompt` file;
  it had already found 8 session files across the 39 harness stores
  (scan took 429 s).
- `bundle --dry-run` #2 (valid system-prompt file): killed by my own
  `timeout 540` at 540 s — the full-store scan takes >9 min. Lesson: use
  `--no-discover --session <file>` instead (8 s).
- Session files located directly:
  - builder (mine): `~/agents/agent-2f8a03f3-862d-4ab7-912c-c69309079846/sessions/2f8a03f3-862d-4ab7-912c-c69309079846.jsonl`
    (created 2026-11-16T23:52:27Z, after the competition opened).
  - parent coordinator: `~/agents/agent-d5a19065-4625-43f4-9be4-e758a1cc16f9/sessions/d5a19065-4625-43f4-9be4-e758a1cc16f9.jsonl`
    (created 2026-11-16T23:49:24Z, 177 lines; its first user message is
    Queue 6, not this bounty — it is the Wave 2 coordinator's session).
- Muse usage rows (`muse.db`, exact query from `bundle --help`):
  only 1 row exists so far, the coordinator's
  (`message 70c5f542-…`, input 2,278,430, output 12,444,
  model `ipnext/avocado-5.17-v2`). ZERO rows for the builder session yet:
  bundle's own warning says a subagent's row is written only once its
  run completes, and the top-level agent must re-run the SELECT and
  re-bundle after that. The DB also records both agents' model as
  `ipnext/avocado-5.17-v2` (not `muse-spark-1.3`); the bundle was run
  with `--model muse-spark-1.3 --provider meta` (the runtime identity)
  and the usage row keeps its own model — whether make.fun's pricer
  knows `ipnext/avocado-5.17-v2` is UNVERIFIED and must be watched at
  submit time.
- No harness version is recorded anywhere accessible (no version column
  in `agent.agents` / `agent.sessions`, no CLI to ask). The bundle was
  run with `--harness-version 1.3` (the only version number attached to
  this stack in context, "Muse Spark 1.3") — STATED, NOT VERIFIED;
  correct it at re-bundle if the real harness version is known.
- Real bundle (DRAFT) written:
  `transcripts.tar.gz`, 253,794 bytes,
  sha256 `95b3f418012575efce970eb96af973378e8bd63ea8a39241f1f66bf603408182`
  - 2 sessions: builder 75 turns / 277 lines; coordinator 14 turns /
    55 lines kept (122 earlier records cut — before the repo was named).
  - 1 usage row attached (coordinator's); 0 for the builder (see above).
  - Run record: prompt handed-off (via "Wave 2 coordinator subagent
    d5a19065 spawned this builder from the user's Wave 2 order"),
    instruction files collected (1: `~/AGENTS.md`), MCP none,
    system prompt in-session.
  - Redactions by bundle: 4 email addresses, 178 home paths,
    123 usernames, 1 credential.
  - Flags: `--no-discover --session <builder jsonl> --muse-usage
    /tmp/tr-muse-usage.jsonl --harness Muse --harness-version 1.3
    --provider meta --model muse-spark-1.3 --effort default --mcp none
    --system-prompt /tmp/tr-sysprompt.txt --prompt-file /tmp/tr-prompt.txt
    --prompt-origin handed-off`.
- The bundle is a DRAFT: it does NOT include this BUILD-LOG's final
  lines or the final commit (written after bundling), and it lacks the
  builder's usage rows. The parent MUST re-run the SELECT + bundle
  after this run completes, then re-run `check` with the fresh bundle,
  before submitting. `transcripts.tar.gz` is git-ignored, never committed.

## `makefun check` — 2026-11-17 ~00:17 WIB (commit `755ee21` + draft bundle)

- `node <kit>/cli/makefun.mjs check --kit <kit> --repo <entry>
  --transcripts <entry>/transcripts.tar.gz`
- **15 passed, 0 failed, 6 skipped, 3 checked at submission** —
  `ok: every check run here passes; the rest are the site's at submission`
  - competition_open, snapshot (15 files, 616,849 B), license_osi (MIT),
    entrypoint (main.ts, node22), repo_created_after_open (root
    `ea5094e`, drand round 34023303, 2026-11-16T23:54:33Z): pass.
  - transcripts_present: 2 agent sessions parsed (2 muse-app).
  - transcripts_recorded, thinking_wellformed (219 messages, 99 tool
    results each answering its own call; 1 tool call without a result —
    a hosted tool, or still running when taken), thinking_in_window
    (all 221 timestamps in window): pass.
  - thinking_reproduces: 680 of 1,628 counted lines (41.7%, need 30%);
    entrypoint main.ts has 19 authored lines.
  - thinking_turns: 77 assistant turns (need 5).
  - thinking_priced: 1 usage record, each priced to a known model
    (the `ipnext/avocado-5.17-v2` row PRICES — the model question is
    answered for this row).
  - thinking_revised: lib/model.ts from 12 write/edit messages,
    main.ts from 2 (need 2).
  - thinking_fresh: all 680 credited lines from sessions showing the
    anchor `makefun anchor ea5094edeca612f79b50c2290f469a08d83b75a7 round
    34023303` before their first counted write.
  - transcripts_run_record: matches the upload (handed-off prompt,
    agent facts, system prompt in-session, instruction files collected,
    no MCP servers).
  - At submission (site-side): repo_owner (wallet must be on the
    account), source_bundle, duplicate_snapshot.
  - Note: gitleaks is not installed here — transcripts were redacted
    with the built-in rules only; the site also runs gitleaks.
- This check ran against commit `755ee21`. The commit after it adds
  only this log section (no code change). Before submitting, the parent
  MUST re-run the SELECT + bundle (builder usage rows attach once this
  run completes) and re-run `check` with the fresh bundle + final HEAD.

## NOT done (by design — parent's call)

- NO `makefun submit` run. NO GitHub push (source-bundle route; no repo
  created on GitHub). No other bounty touched.
