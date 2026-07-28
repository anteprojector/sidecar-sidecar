# Opt-in merge strategies

Status: design musing, 2026-07-28. Not committed to building this; captured
while the thinking was fresh.

Today every automated sync resolves conflicts one way: fork-files. That is the
right default — nothing blocks, nothing is lost — but some files have a
*correct* automatic answer (two machines appending to a journal) and some have
no acceptable automatic answer at all (billing records). This is the design
for letting paths opt into different strategies.

## Prior art, briefly

The two postures in the wild are git-auto-sync (rebase-only; on conflict,
abort the sync and notify — safe, but the repo stalls) and it3xl/git-repo-sync
("newest wins" by ref manipulation — never blocks, but the loser's work is
silently displaced until they manually recover). Neither found a way to
actually auto-merge; sidecar's fork-files is a third posture, closest to
Dropbox/Syncthing "conflicted copy" files but with a committed manifest.
Per-path strategies would let a repo mix all three postures deliberately.

## The one hard constraint: convergence

Every machine runs the same merge sweep independently. If two machines
resolve the same conflict differently, main ping-pongs forever. So a strategy
must be a **pure function of the two conflicting versions plus committed
config** — never of "which machine am I" or which side is checked out here.
Consequences:

- Strategy declarations live in shared, committed state — never the
  machine-local state dir.
- Inbox merge order must be deterministic (sort branches lexically) so
  direction-dependent strategies resolve identically everywhere.
- Side-wins strategies are named by *role* (`canonical` / `incoming`), not
  direction (`ours` / `theirs` would mean different things on different
  machines).

## The strategy set (closed — no external resolvers, see below)

- **`fork`** — today's behavior; stays the default. Right for anything a
  human or agent authored.
- **`union`** — concatenate both sides' additions (git's built-in
  `merge=union` semantics). Highest-value addition for this domain:
  append-only agent journals, NDJSON event logs, TODO lists. Most "conflicts"
  in note streams are two machines appending, where union is simply correct.
- **`newest`** — last-writer-wins by snapshot commit timestamp,
  deterministic tiebreak on commit hash. For single-value state files
  (current-focus pointers, cursors, generated indexes) where any recent
  version is fine and forked copies are noise.
- **`canonical` / `incoming`** — deterministic side-wins by role: `canonical`
  keeps the version already on main; `incoming` takes the inbox's. For files
  one owner is authoritative over. (git-repo-sync's "conventional" strategy,
  scoped to paths instead of branches.)
- **`block`** — never auto-resolve. The conflicting inbox branch is left
  unmerged (inboxes persist, so nothing is lost), the daemon keeps merging
  *other* inboxes and repos — block must not wedge the sync — and the stall
  is surfaced via `sidecar status`, the health heartbeat, and a local system
  notification. A human resolves manually; the next sweep proceeds. For paths
  where a wrong auto-resolution is worse than waiting.

Deliberately absent:

- **`merge`** — git already does trivial 3-way merges before any strategy
  fires; by the time one runs, the overlap is real.
- **`resolver` (external command) — decided against.** A resolver command in
  committed config is code execution on every machine that syncs the repo;
  the mitigation (per-machine allowlists, daemon approval prompts,
  skip-and-flag states) is real UX weight guarding a hole that shouldn't
  exist. Dropping it makes the strategy set closed and every strategy a pure
  function of blobs + commit metadata: *nothing in the system executes
  anything declared in shared content.* The use case (LLM / structured JSON
  merges) is still reachable through the fork path, post-hoc: manifests
  preserve both versions with oids and hashes, so a future `sidecar resolve`
  command — or any agent reading `.sidecar-conflicts/` — can reconcile forked
  pairs later as an ordinary working-tree edit, a deliberate local action
  rather than fleet-wide auto-execution. The reserved `--llm` flag's message
  ("reserved for a configured resolver") should eventually be reworded toward
  this post-hoc model so it doesn't promise the thing we chose not to build.

## Declaration: three layers

### 1. In-file pragma (most local, self-describing)

```
# sidecar: merge=union
```

Scanned from the first ~1KB / 16 lines of a conflicting file with a marker
regex, regardless of comment leader (`#`, `//`, `<!-- -->` all work — no
per-language comment parsing). Pragmas are the *best* fit for the convergence
constraint: the strategy becomes a pure function of the versions alone, with
no shared-state distribution problem. They also self-describe to anyone
reading the file — including agents, who can adopt a strategy at
file-creation time without touching shared config or racing a push.

Rules:

- Both current sides (stages 2 and 3) must declare the same strategy;
  disagreement — including one side removing the pragma — falls through to
  the next layer. A dispute over the strategy itself is exactly a case for
  forking.
- Pragmas are file content, the least-trusted input in the system, so they
  may select only content-level strategies (`union`, `newest`, `fork`) and
  may never downgrade a config-declared `block`.
- Cost is negligible: only files that actually conflicted are scanned.

### 2. Filename marker

`events.union.ndjson` → union; implemented as nothing more than built-in glob
rules (`**/*.union.*`, `**/*.newest.*`). Covers the two cases pragmas can't —
JSON (no comments) and binary files — and self-describes in a directory
listing.

### 3. Rules file, in the sidecar checkout (not `.sidecar`)

```toml
default = "fork"

[[rule]]
path = "journal/**/*.md"
strategy = "union"

[[rule]]
path = "state/**"
strategy = "newest"

[[rule]]
path = "billing/**"
strategy = "block"
```

Gitignore-style globs, **last match wins** — semantics every git user already
has intuitions for. Per-dir and per-file fall out of the same mechanism.

Why not `.sidecar`? Skew. `.sidecar` is committed in the *parent* repo, which
syncs at human pace, while the sidecar checkout syncs every minute — a rules
change in `.sidecar` could take days to reach other machines, during which
they resolve conflicts differently. A rules file *inside the sidecar
checkout* (say `.conflict-rules.toml`) distributes through sidecar's own
sync channel and converges within a minute. With resolvers gone it carries
only strategy names — pure data, no security dimension. Bootstrap wrinkle:
the rules file needs a pinned, hardcoded strategy for its own conflicts — pin
`newest` (forking it would silently deactivate everyone's rules).

### Precedence (most-local wins, except where trust forbids)

1. Config `block` rules — not overridable from below
2. In-file pragma (safe strategies only; both sides must agree)
3. Filename marker
4. Rules-file globs, last match wins
5. Default (`fork`)

## Implementation notes

- There is already a single choke point: `forkConflicts` iterates unmerged
  paths with both sides' blobs in hand during the inbox sweep. Per-path
  strategies are a dispatch table in front of that loop, not a merge rework.
- Keep dispatch sidecar-side rather than compiling to `.gitattributes` merge
  drivers: custom drivers need per-machine `.git/config` wiring anyway,
  `newest` needs commit metadata git doesn't hand to drivers, and one code
  path beats two. (gitattributes `merge=union` could be honored as an
  implementation detail, but users declare in sidecar's own layers.)
- Every non-fork resolution still writes a `.sidecar-conflicts/` manifest
  entry — `resolved_by` plus `declared_by: "pragma" | "filename" | "rule" |
  "default"` and both oids — so auto-resolution is auditable and a surprising
  outcome is diagnosable after the fact.
- Old-CLI compat: unknown config keys are already ignored, so old CLIs fall
  back to fork-everything — safe. State that as the explicit contract.
- Agent-facing docs get a one-liner: "starting an append-only log? put
  `# sidecar: merge=union` at the top or name it `*.union.ndjson`."

## Phasing, if built

1. `union` + the rules-file dispatch (small, git-proven semantics, biggest
   payoff for note streams)
2. Pragma + filename markers
3. `newest`, `canonical`/`incoming`
4. `block` (carries the daemon-UX weight: leave-inbox-unmerged, health
   surfacing, status flag)
5. Post-hoc `sidecar resolve` for forked pairs (the ex-`--llm` story)
