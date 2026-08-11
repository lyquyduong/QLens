# QLens

**Every limit, in focus.**

QLens manages your AI provider accounts and tracks how much of each one you have left — across
116 providers, on macOS, Linux and Windows, as a desktop app, a terminal UI, and a scriptable CLI.

The question QLens answers is not "here are forty numbers". It is **how much do I have left, and
when does it reset.**

*Sketch of where this is going — `qlens quota` renders the windows today; the burn column and the
`status` roll-up are `core::analysis` output that no face draws yet.*

```
$ qlens status
PROVIDER          PLAN          WINDOW          USED            RESETS IN   BURN/H
claude            Claude Code   session (5h)    ███████████░░  87%   2h14m    12.4%
                                weekly (7d)     ███████░░░░░░  48%   4d02h     1.1%
codex             Plus          session (5h)    ████░░░░░░░░░  31%   4h02m     6.8%
kiro              Free          monthly         ██░░░░░░░░░░░  12%  23d11h     0.4%

3 of 4 connections below their alert threshold · soonest reset claude session (5h) in 2h14m
```

---

## Status

Early development, and **v0.1.1 is installable on macOS** — see below. Milestones land in order;
each one is gated on a verification step rather than a "looks done" judgement.
[`docs/ROADMAP.md`](docs/ROADMAP.md) carries the detail: what is done, what each decision rests on,
and what is blocked on a product call.

The source is not published here. This repository carries the releases; the code lives elsewhere,
which is why there is nothing to clone.

| Milestone | Scope | State |
|---|---|---|
| **M0** Foundation | cargo workspace, core crate, toolchain pin | ✅ |
| **M1** Registry | 116 provider descriptors as data, with a fidelity gate | ✅ |
| **M2** Auth | API key · OAuth PKCE · device code · refresh | ✅ |
| **M2.5** First light | one provider end to end, from a terminal | ✅ |
| **M3** Quota | per-provider quota sources, adaptive poller, history | 🟡 all 16 ids · mappings unverified against live accounts |
| **M4** Daemon | one poller, many clients, over a local socket | 🟡 verified live on Unix · Windows transport written and type-checked, never executed |
| **M5** Desktop | Tauri 2 shell, tray, notifications | 🟡 **ships in v0.1.1** — launched, and its window confirmed working · ad-hoc signed, not notarised · nothing starts the daemon for it |
| **M6** TUI + CLI | Ratatui mirror, clap, shell-prompt output | 🟡 CLI is a daemon client, nine commands · TUI draws overview, catalogue and detail |
| **M7** Analytics + ship | burn rate, cost, signed installers | 🟡 burn rate + projection · **v0.1.1 macOS universal binaries published, ad-hoc signed** · no Developer ID, no notarisation, no Homebrew tap, no Windows or Linux artefact |

### Install — macOS, v0.1.1

The release carries three binaries as one universal (arm64 + x86_64) tarball. **`qlensd` is not
optional**: every face is a client of it, and nothing starts it for you yet.

```bash
V=0.1.1
curl -fsSLO https://github.com/lyquyduong/QLens/releases/download/v$V/qlens-$V-macos-universal.tar.gz
curl -fsSLO https://github.com/lyquyduong/QLens/releases/download/v$V/SHA256SUMS
shasum -a 256 -c SHA256SUMS          # verify before you run anything

tar xzf qlens-$V-macos-universal.tar.gz
sudo install -m755 qlens qlensd qlens-tui /usr/local/bin/

qlensd &                             # leave it running
qlens providers --quota-only
qlens connect antigravity            # ⚠ real OAuth, opens a browser
qlens quota
```

The binaries are **ad-hoc signed, not notarised**. `curl` does not set the quarantine attribute, so
they run as downloaded; a `.dmg` fetched through a browser would be a different matter, and
[`docs/running.md`](docs/running.md) covers that case.

### Install — the desktop app

New in v0.1.1, and **ad-hoc signed rather than notarised**, which decides how you should fetch it.

```bash
V=0.1.1
curl -fsSLO https://github.com/lyquyduong/QLens/releases/download/v$V/qlens-$V-macos-app-universal.tar.gz
tar xzf qlens-$V-macos-app-universal.tar.gz
mv QLens.app /Applications/
```

**Prefer the tarball over the `.dmg`.** `curl` does not set the quarantine attribute, so the app
opens normally. A `.dmg` downloaded through a browser *is* quarantined, and Gatekeeper refuses an
un-notarised app — you would have to go to *System Settings → Privacy & Security → Open Anyway*.
The `.dmg` is published for people who want it; the tarball is the path that just works.

**The app does not start `qlensd`.** Open it without one running and you get an empty window with
no explanation. That gap is real and it is the next thing worth closing.

### The signed update manifest

Every release also carries `latest.json` and `latest.json.minisig` — a minisign-signed manifest
naming the CLI tarball and its SHA-256, verifiable with the key compiled into every QLens binary:

```bash
minisign -Vm latest.json -P RWT6p3c9qDJv/3fb0L30JlkNT+SgYYdhaxIiXOLZaXbOcJPc4mKDH25p
```

Nothing consumes it yet — `qlens update` is still to come. Publishing it from the first release
means the upgrade path has something to upgrade *from*.

### Building it instead

**`qlensd` owns polling, the store, the keychain and every authorisation flow; `qlens`, `qlens-tui`
and the web UI are clients.** Nothing spawns the daemon for you — start it first.
[`docs/running.md`](docs/running.md) is the guide.

```bash
cargo run -p qlens-daemon                             # qlensd — everything else needs it
cargo run -q -p qlens-cli -- providers --quota-only   # the 16 QLens can poll
cargo run -q -p qlens-cli -- connect antigravity      # ⚠ real OAuth, opens a browser
cargo run -q -p qlens-cli -- quota                    # the daemon's last reading — no upstream call
```

Unix is the only platform this has *run* on. The Windows named-pipe transport is written — a
658-line `#[cfg(windows)]` module in `crates/daemon/src/listener.rs` — and `tools/check-windows.sh`
type-checks it against the real `x86_64-pc-windows-msvc` target, but nobody here has the machine to
execute it. The `compile_error!` that remains fires only on a target that is *neither* Unix nor
Windows.

Tokens go to the OS keychain; the SQLite row beside them holds a *reference* and never a secret,
and a test scans every text value in every table to prove it. Nobody else on the machine can reach
the daemon — the socket is `0600` inside a `0700` directory and the peer's uid is checked at
`accept`. VS Code tasks and debug configs are committed in `.vscode/`.

---

## Architecture

Three faces, one core. A behaviour has exactly one implementation, and the GUI, TUI and CLI are
all thin clients over it.

```
                        ┌──────────────┐
              ┌────────▶│   desktop    │  Tauri 2 · React 19 · Tailwind 4
              │         └──────────────┘
┌──────────┐  │         ┌──────────────┐
│  daemon  │──┼────────▶│     tui      │  Ratatui · Crossterm
│  (axum)  │  │         └──────────────┘
└────┬─────┘  │         ┌──────────────┐
     │        └────────▶│     cli      │  clap · JSON · exit codes
     │                  └──────────────┘
     ▼
┌──────────────────────────────────────────────────────────┐
│  core                                                    │
│  registry · auth · quota · scheduler · store · secrets   │
└──────────────────────────────────────────────────────────┘
```

**Why a daemon.** Quota endpoints are rate-limited, and some of them aggressively. If each client
polled for itself, opening the GUI while the TUI was already running would double the request rate
and earn a 429 — punishing the user for using the product. One scheduler owns every upstream call;
everyone else subscribes. It also means quota history keeps accruing while no window is open,
which is what makes burn-rate and projections possible at all.

**Why the registry is data.** All 116 provider descriptors are inert TOML — endpoints, headers,
models, auth shape. No descriptor can imply behaviour. Provider-specific logic lives in the module
that reads the descriptor, keyed off `id`. This is what keeps 116 providers from becoming 116
special cases, and it is enforced by a test rather than by good intentions.

**Where secrets live.** In the OS keychain — macOS Keychain, Windows Credential Manager, libsecret
— never in the database. The local store keeps metadata and a reference, so a leaked database file
is not a leaked set of credentials.

---

## Layout

```
crates/core/        domain: registry · auth · quota · scheduler · store · secrets
crates/daemon/      axum over a unix socket / named pipe
crates/desktop/     Tauri 2 shell
crates/tui/         Ratatui
crates/cli/         clap
registry/           116 × <provider>.toml — provider descriptors (data only)
data/               generated reference data (pricing tables, …)
design/             design-token contract and the Stitch handoff brief
docs/               implementation specs written before the code they govern
ui/                 Tauri frontend (React 19 · TypeScript · Tailwind 4 · shadcn/ui)
xtask/              repository automation: data ports, generators, verification gates
tools/              one-off Node scripts for reference-data extraction
```

---

## Development

```bash
cargo test                          # workspace
cargo clippy --all-targets
cargo xtask --help                  # repository tasks
node tools/validate-tokens.mjs      # design tokens against their schema
```

### The design-token contract

`design/tokens.json` is the single source that generates both the GUI theme (Tailwind CSS
variables) and the TUI theme (`ratatui::style::Color`), so one malformed token file breaks two
surfaces at once — and it arrives from an external design tool rather than through code review.
`design/tokens.schema.json` is therefore written **before** the design, enumerating every token and
every state that must exist, and `tools/validate-tokens.mjs` checks incoming files against it.

The validator implements the JSON Schema subset the contract uses rather than taking a dependency,
and refuses to run if the schema grows a keyword it does not implement — a skipped constraint is
worse than no validation, because it reads as a pass.

That strictness has already paid for itself: it surfaced that `prefixItems` sitting beside `items`
exempted `quota.bands[0]` from the band schema entirely, so the one band that styles a 0% reading
could have omitted every required field. Fixed by nesting the constraint inside `allOf`.

### Fidelity gates for ported data

Two datasets — the 116 provider descriptors and the model pricing table — were transcribed
mechanically from a reference implementation. By hand they would attract exactly the kind of
quiet, single-field error that surfaces months later as one provider mysteriously failing, or as
every cost figure being wrong by an amount nobody can spot by eye.

Mechanical transcription is only worth trusting if it is checkable, so it is checked:

```bash
cargo xtask sync-reference          # clone/refresh the reference, re-dump, classify what changed
cargo xtask sync-reference --apply  # …re-port both datasets and run both gates below
```

which composes these, each runnable on its own:

```bash
node tools/dump-9router.mjs --source /path/to/reference   # snapshot provider descriptors
node tools/dump-pricing.mjs --source /path/to/reference   # snapshot pricing tables
cargo xtask port-registry                                 # snapshot -> registry/*.toml
node tools/gen-pricing-toml.mjs                           # snapshot -> data/pricing.toml
cargo xtask verify-registry                               # prove nothing was lost
cargo xtask verify-pricing
```

Each gate has two halves, and both matter: a canonical-form diff against the snapshot proves no
data was lost, and a typed load through `crates/core` proves the schema actually describes what
was written.

Both gates have already earned their keep:

- The registry's typed-load half caught `hasOAuth` silently falling through to the untyped
  catch-all for **nine** providers, because `camelCase` of `has_oauth` is `hasOauth`. No amount of
  reading would reliably have found that.
- `verify-pricing` asserts pattern **order** separately from pattern content, because pricing
  resolution is first-match-wins: swapping two adjacent patterns reprices models without changing
  a single rate value. A pure value diff would call that a pass.

`port-registry` will not overwrite an existing descriptor without `--force`. Once verified,
`registry/` and `data/` are the source of truth, the reference checkout is no longer needed, and
the snapshots are disposable — which is why they are gitignored.

Re-running the transcription against a newer reference is specified in
[`docs/reference-sync.md`](docs/reference-sync.md). Two properties of it are worth naming here: a
regeneration carries the declared divergences (below) across, comments included, so an unchanged
reference re-ports **byte-identically**; and a change in one of the pricing *behaviour probes* —
answers recorded by running the reference, not read off it — blocks the re-port, because
re-transcribing data cannot fix a change in resolution logic.

### Diverging from the reference on purpose

A gate that forbids all change would freeze this data to a copy of someone else's decisions, and
a gate people cannot live with gets switched off — at which point it protects nothing. So
`verify-registry` allows differences that are **declared with a reason** in
`data/registry-divergences.toml`, and enforces the rule in both directions: an undeclared
difference fails, and a declaration that no longer matches a real difference *also* fails, so the
file cannot rot into a list of stale excuses. Declared divergences are printed with their
reasoning on every run, so the justification stays in front of whoever reads the output.

The three declared today all restore one property: the reference establishes that quota endpoints
come from the registry (`U(id) => PROVIDERS[id]?.usage`) and then three handlers escape it by
holding their endpoint as a module constant — `kimi`, `deepseek`, and grok-cli's gRPC
SuperGrok-pool call. Those endpoints now live in the descriptors, so QLens's quota poller really
does read every endpoint from one place.

### Measuring before typing

The descriptor schema was derived by measuring every real descriptor (112 at the time, 116 today), not by reading the
reference's own type documentation, which turned out to be incomplete. Three unions came out of
that measurement, each of which would have been a silent bug if the schema had been guessed:

| Field | Reality |
|---|---|
| `oauth.scopes` | `string \| array` — `github` uses a string, the other four use arrays |
| `transport.retry.<status>` | `integer \| table` — both `429 = 2` and `429 = { attempts = 2 }` occur |
| `models[].rateMultiplier` | fractional (0.6–2.4), while every other number in the registry is integral |

The same approach applies to pricing, where the reference's glob matcher is anchored at both ends
and case-insensitive while its two exact-match levels are case-sensitive. That inconsistency is
observable — `MiniMax-M3` and `minimax-m3` resolve to different prices — so it is preserved
deliberately and pinned by a test rather than quietly "fixed".

---

## The data today

**`registry/` — provider descriptors**

| | |
|---|---|
| descriptors | **116** |
| with a transport | 81 |
| with an OAuth flow | 20 |
| flagged quota-capable | 19 |
| with declared quota endpoints | 17 |
| categories | 74 apikey · 18 oauth · 18 freeTier · 4 free · 2 webCookie |

Adding a provider means adding one TOML file. Only a provider whose upstream is genuinely unusual
needs code.

**`data/pricing.toml` — cost resolution**

97 canonical model rates, 2 provider override tables (111 entries), and 51 ordered glob patterns, resolved
first-match-wins: provider override → canonical model → glob pattern. All rates are US dollars per
million tokens.

---

## License

MIT
