# Merge strategies and block rules

Status: design musing, revised 2026-07-28. Not committed to building this;
captured while the thinking was fresh. Decisions marked **decided** were
settled in discussion; everything under Open questions was not.

Today every automated sync resolves conflicts one way: fork-files. That is the
right default — nothing blocks, nothing is lost — but some files have a
*correct* automatic answer (two machines appending to a journal) and some
content should never leave the machine at all. This is the design for both.

Two separate features share one declaration mechanism:

- **Merge strategies** — how a conflict at a path is resolved. `fork`,
  `union`, `lww`.
- **Block rules** — paths that never sync. **Not a merge strategy.** Additive
  and non-removable: a block declared anywhere in the chain cannot be
  disabled deeper in the tree.

## Prior art, briefly

The two postures in the wild are git-auto-sync (rebase-only; on conflict,
abort the sync and notify — safe, but the repo stalls) and it3xl/git-repo-sync
("newest wins" by ref manipulation — never blocks, but the loser's work is
silently displaced until they manually recover). Neither found a way to
actually auto-merge; sidecar's fork-files is a third posture, closest to
Dropbox/Syncthing "conflicted copy" files but with a committed manifest.
Per-path strategies let one repo mix all three postures deliberately.

## The spine: convergence

Every machine runs the same merge sweep independently. If two machines
resolve the same conflict differently, main ping-pongs forever. So every
strategy must be a **pure function of the two conflicting versions plus
shared, committed rules** — never of "which machine am I", which side is
checked out here, or which order this machine happened to merge inboxes in.

Everything below follows from that, including the two non-obvious
requirements: union must be commutative and idempotent, and rules must travel
on the same sync channel as the content they govern.

## Declaration: nested `.sidecar-rules` files

**Decided:** `.sidecar-rules`. It joins the existing `.sidecar` /
`.sidecar-conflicts/` family, sorts adjacent to `.sidecar` in a listing, and
leaves room for future siblings.

Rules nest like `.gitattributes`: the effective ruleset for a path is every
`.sidecar-rules` found walking from the file's directory up to the parent
repo boundary (in standalone mode, the repo root). Deeper files take
precedence over shallower ones; within a file, last match wins. Patterns are
gitignore-style globs, anchored relative to the containing file's directory.

### Format: line-oriented, not TOML

```
# strategy rules
journal/**        merge=union
state/**          merge=lww
notes/**.md       merge=fork

# block rules — additive, cannot be disabled deeper
secrets/**        block
*.pem             block
```

`<pattern> <attribute>`, where attribute is `merge=fork|union|lww` or
`block`. Block is an attribute rather than a `merge=` value, so it is
structurally not a merge strategy.

**Why not TOML, despite `.sidecar` being TOML:** the rules file has to survive
being union-merged (see below), and a line union of two TOML files that both
set the same key produces duplicate keys and an unparseable file. Any set of
lines is a valid line-oriented rules file, so union is structurally safe with
no content-aware special case. It also makes last-match-wins natural and
diffs cleanly.

### Not in `.sidecar` — **decided**

In **standalone** mode (`path = "."`) `.sidecar` and `.sidecar-rules` are
siblings in the same synced directory, so allowing both just splits rules
across two files for no gain. In **nested** mode `.sidecar` lives in the
*parent* repo, outside the sync channel entirely: it propagates only when a
human pushes and pulls the parent, while its subject matter propagates every
minute. That is invisible rule skew, and skewed rules break convergence.

The checkout-root `.sidecar-rules` is the root of the chain in both modes.
`.sidecar` keeps its single job: identity and transport.

### Rejected: in-file pragmas and filename markers

An earlier draft had `# sidecar: merge=union` pragmas and `*.union.ndjson`
filename markers. Both were dropped as footguns. The pragma's "both sides
must agree" rule was papering over the fact that untrusted file content —
writable by any machine or agent — was steering the merge, and a strategy
that silently changes when someone edits a comment is not a strategy anyone
can reason about. Filename markers make a file's identity load-bearing:
renaming changes semantics, and the marker is invisible to anyone reading the
file itself.

## Strategies

### `fork` — the default

Today's behavior: both versions written out as sibling files, original
removed, a manifest written to `.sidecar-conflicts/`. Right for anything a
human or agent authored, and the fallback whenever another strategy cannot
apply.

### `union` — commutative, idempotent, content-blind

Git's built-in `merge=union` concatenates ours-then-theirs and never dedupes,
which is **not** safe here: two machines merging the same inboxes in different
orders, or against different partially-merged bases, produce different blobs,
push, re-conflict, and re-union — accumulating duplicate lines forever.

The fix is to make union a *set* operation with a total order that does not
depend on merge direction or merge path:

- **Order key: per-line introduction time.** Git blame attributes a line to
  the commit that actually introduced it, and that attribution survives
  merges — so it is invariant to which machine merged what in which order.
  Sort conflicting-hunk lines by `(introduction commit time, commit oid,
  index within that commit's addition)`. All three components are
  deterministic; the oid tiebreak covers equal timestamps across machines.
- **Dedupe by line identity**, not by text: a line is identified by its
  introduction commit plus its index within that commit's addition. Re-unioning
  already-unioned content is therefore a no-op (idempotence), while two
  genuinely distinct occurrences of the same text — repeated separators, a
  log line that legitimately recurs — both survive.
- **Only conflicting hunks are reordered.** Non-conflicting regions merge
  normally and keep their position, so a hand-edit elsewhere in the file is
  untouched.
- **Fall back to fork** for non-text content or when blame data is
  unavailable.

Note what this deliberately does *not* do: it never inspects content
semantics, parses timestamps out of log lines, or sorts lexically. It is
metadata-driven and works on any line-oriented text.

Considered and rejected: keying order on each side's *tip* commit time. It is
much cheaper (no blame) and is commutative for a single pair, but it is not
order-independent across a multi-inbox sweep — merging inbox B then C versus C
then B re-derives keys from a different base and diverges.

### `lww` — last write wins

Whole-file granularity — per-hunk lww has no coherent meaning. Winner is
decided by committer timestamp, tiebroken deterministically on commit oid.

The honest caveat: this trusts machine clocks, so a machine with a fast clock
always wins. Health heartbeats already carry per-machine data and could detect
skew. For single-value state — cursors, current-focus pointers, generated
indexes — where any recent version is fine and forked copies are pure noise.

### Delete/modify conflicts

Undefined for all three strategies, so hardcode it: **deletion never wins over
content**, regardless of the declared strategy. Keep the surviving version and
log it to the manifest. A notes repo should not lose content to a race.

### Not included

`merge` (git already does trivial 3-way merges before any strategy fires — by
the time one runs, the overlap is real), `canonical`/`incoming` (deferred, not
rejected), and external resolver commands.

**Resolvers were decided against.** A resolver command in committed config is
code execution on every machine that syncs the repo; the mitigation
(per-machine allowlists, daemon approval prompts, skip-and-flag states) is
real UX weight guarding a hole that shouldn't exist. Dropping it keeps the
strategy set closed and every strategy a pure function of blobs plus commit
metadata: *nothing in the system executes anything declared in shared
content.* The use case — LLM or structured JSON merges — is still reachable
post-hoc: fork manifests preserve both versions with oids and hashes, so a
future `sidecar resolve` can reconcile forked pairs later as an ordinary local
edit rather than fleet-wide auto-execution. The reserved `--llm` flag's
message ("reserved for a configured resolver") should eventually be reworded
toward this model so it stops promising the thing we chose not to build.

## Block rules

Additive and non-removable. There is no negation syntax — a block matching
anywhere in the chain wins, and no deeper file can undo it. That is the whole
point: protection can only ever be added.

- **Enforced at snapshot, never at merge.** A merge-time filter is useless —
  the content is already committed to the inbox and pushed, so the secret has
  already left the machine. Block must prevent the `git add` entirely.
- **Skip and report; do not error — decided.** Erroring would wedge the
  daemon: one constantly-rewritten blocked file (a log, an `.env`) would fail
  every sync forever, and the sync must keep making progress on everything
  else. This mirrors the existing soft-request philosophy in the lock
  handling. Silence is equally wrong — "I thought that was backed up" is the
  one surprise you cannot afford. So: skip the path, complete the sync,
  surface a blocked-path count in `sidecar status` plus an event-log entry.
  Not a notification per change. A manual command that explicitly targets a
  blocked path may error, since that is a demand rather than a background
  pass.
- **Block cannot retroactively unpublish.** Adding a rule for a path that
  already synced leaves the content in remote history. Say so loudly rather
  than implying protection that isn't there.
- **Old CLIs will not honor blocks — accepted.** The pinned project-local CLI
  invariant means a machine may run a version that has never heard of
  `.sidecar-rules` and will happily snapshot and push blocked content. There
  is no way to make an old binary respect a new rule. Detection is still worth
  it: heartbeats already carry per-machine versions, so status could flag
  "block rules present; machine X runs a CLI too old to honor them."
- **Why not just `.gitignore`?** Gitignore permits `!` negation and
  per-checkout overrides, which is exactly the additivity block exists to
  guarantee. Implementation may still use git's exclude machinery underneath.
- **Machine-local block is out of scope — decided.** That is just
  `.gitignore`, which already works.

## The rules file's own conflicts

`.sidecar-rules` is synced content and can itself conflict. Pinned strategy:
**early validation, then `union`** — no content-aware special case needed,
because the line-oriented format makes any union result structurally valid.

This is fail-safe in the direction that matters: union never deletes, so a
block rule can never silently vanish in a merge. Contradictory `merge=` lines
for the same pattern both survive, and last-match-wins resolves them
deterministically because union's ordering is deterministic — effectively the
later-declared rule wins. The accepted downside is that *deleting* a rule can
be undone by any machine still carrying the old version; it re-deletes once
the fleet converges.

Validation is where safety comes from. Reject invalid files at write time, and
fail closed on read: a malformed `.sidecar-rules` must not degrade to "no
rules", since that would silently deactivate a block. Refuse to sync that repo
with a clear error, matching the redaction filter's existing fail-closed
stance.

## Implementation notes

- There is already a single choke point for strategies: `forkConflicts`
  iterates unmerged paths with both sides' blobs in hand during the inbox
  sweep. Per-path strategies are a dispatch table in front of that loop, not a
  merge rework. Block, by contrast, belongs in snapshot.
- **Read rules from the canonical branch state being merged into**, not from
  the working tree, or determinism leaks.
- **Deterministic inbox merge order** (sort branches lexically) — necessary
  regardless of union's internal ordering.
- Keep dispatch sidecar-side rather than compiling to `.gitattributes` merge
  drivers: custom drivers need per-machine `.git/config` wiring anyway, `lww`
  needs commit metadata git doesn't hand to drivers, and one code path beats
  two.
- Every non-fork resolution still writes a `.sidecar-conflicts/` manifest
  entry — `resolved_by` plus the declaring file and line — so auto-resolution
  is auditable and a surprising outcome is diagnosable after the fact.
- **Ship `sidecar rules <path>`** alongside the feature: print the effective
  strategy and block status for a path plus the file and line that declared
  it, like `git check-attr` / `git check-ignore -v`. Nested rule systems are
  miserable to debug without it.
- Old-CLI compat: an unknown rules file is simply unread, so old CLIs fall
  back to fork-everything and no blocks — safe for strategies, accepted for
  block.

## Open questions

- **Is blame fast enough?** Union needs per-line introduction data for every
  conflicting file. Scoped to conflicting hunks (`git blame -L`) on
  notes-sized files it should be fine, but it is unmeasured.
- **Forward compat vs fail-closed on unknown attributes.** A malformed line
  should fail closed. But a *well-formed* line with an attribute a newer CLI
  understands and this one doesn't — fail closed (safe, but a newer machine
  can halt older ones fleet-wide) or ignore-with-warning (evolvable, but a
  future block variant would be silently ignored)? Leaning ignore-with-warning;
  not settled.
- **Rules file cruft.** Union accumulates superseded `merge=` lines. Worth a
  `sidecar rules --prune`, or just tolerable?
- **Inbound content at a blocked path** — flagged, or removed? Blocks are
  fleet-wide, so this only arises with stale rules or old CLIs.
- **`canonical` / `incoming`** — deferred. Deterministic side-wins by role
  (never `ours`/`theirs`, which mean different things on different machines),
  for files one machine is authoritative over.

## Phasing, if built

1. `.sidecar-rules` parsing, nesting, and `sidecar rules <path>` — the
   mechanism, with only `fork` wired up. No behavior change.
2. Block rules at snapshot, with status/event-log surfacing.
3. `union`, including the blame-ordered set semantics and fork fallback.
4. `lww`.
5. Post-hoc `sidecar resolve` for forked pairs (the ex-`--llm` story).
