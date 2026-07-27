# Revisit

Decisions that were right at the time but deserve a second look. Each one is a
question, not a task — the point is to re-examine, not to assume the answer.

- **`SENTINEL.md` — a rendered, clearable alert inbox on the canonical
  branch.** Fleet health currently lives entirely in refs: each checkout
  publishes `sidecar-health/{user}/{id}`, and `sidecar health` reads them back.
  That deliberately keeps the canonical branch pure notes, but it means the
  signal is only visible to someone who runs the command — a human browsing the
  repo on GitHub, or an agent working in a clone, sees nothing. The idea we
  designed and then set aside was a `SENTINEL.md` rendered into the canonical
  branch during `merge`, gated by a config boolean, always present rather than
  appearing only on failure. Its content would be a pure function of the health
  refs, so a merge conflict resolves by regenerating rather than by merging
  text, and the push-race retry in `mergeInboxBranchesAt` handles it for free.
  Worth asking, whenever it comes back up:
  - Is health even the right first tenant? The more interesting version isn't a
    health readout at all — it's a general inbox for *clearable alerts*, where
    anything that wants a human's attention (a redaction false positive worth
    reviewing, a forked conflict nobody resolved, a machine on a stale version)
    files an entry and something later clears it. Health would be one producer
    among several, and the rendering, the clearing protocol, and the "who owns
    an entry" question are the actual design, not the file format.
  - What clears an entry, and who decides? Health entries clear themselves —
    the next good sync overwrites the record. A forked conflict doesn't; it
    needs a person to say "handled". Those are different lifecycles, and a
    single file that mixes them needs an answer for both before it's built.
  - Does it want to be Markdown at all? Rendered Markdown is readable in a diff
    and on GitHub, which is the whole point — but it's also the format most
    likely to tempt someone into hand-editing a generated file.

- **How we bundle chokidar, and whether to ship a single embedded executable
  instead.** Today `build` bundles everything *except* chokidar
  (`--external chokidar`), so the published package is a 156 kB `dist/cli.js`
  plus two runtime `dependencies` — chokidar for watching and smol-toml, which
  *is* bundled. That asymmetry is the thing to look at. `daemon.ts` already
  treats chokidar as optional: `loadChokidar()` wraps the dynamic `import` in a
  try/catch and degrades to interval-only syncing when it's missing, which is
  the right instinct but means a broken install fails silently into a slower
  mode rather than loudly. The reason it's external is that chokidar reaches for
  native fsevents on macOS, which doesn't survive bundling. Worth asking:
  - Could chokidar be bundled with fsevents left external, so the common path
    has zero install-time dependencies?
  - Or drop chokidar entirely for `fs.watch`, which is now recursive on macOS
    and Windows and adequate for one directory per repo?
  - Or go the other way — `bun build --compile` (or Node's SEA) to ship one
    self-contained binary. That kills the dependency question outright and
    removes the Node 20+ requirement, but replaces `npm i -g` with per-platform
    artifacts, a release matrix, and a story for how the daemon updates itself.
    The install path is currently the product's best feature; a binary is only
    worth it if it stays that simple.

- **`dist/cli.js` is committed to the repo.** It's rebuilt by `prepack`, so the
  tarball never depends on the checked-in copy, but every source change carries
  a few hundred lines of bundle diff alongside it. Ask whether the committed
  artifact is still earning its place, or whether it can be gitignored and left
  to the build.

- **Publishing no longer runs the tests.** `prepublishOnly` is just the
  typecheck now, deliberately, because the suite is timing-flaky. That makes CI
  the only gate between a red test and npm. Fine while releases are hand-driven;
  reconsider before automating them.

- **The watch/debounce test is flaky.** It failed once in three runs, then
  passed repeatedly. The suite leans on real timing — one test waits out a
  ~7.7s debounce — so it will fail CI intermittently. Worth making the debounce
  injectable rather than slept-through.

- **The no-redact pragma still can't be written in JSON.** The marker has to
  lead its own line behind punctuation, which no JSON file can do, and redaction
  is configured repo-wide with no per-path override. So a JSON file's only
  escape is turning redaction off everywhere. A path-glob opt-out in `.sidecar`
  would close this without loosening the pragma.

- **A Markdown bullet can disarm redaction.** `- sidecar:no-redact` on its own
  line is indistinguishable from a comment leader, so a doc that lists the
  marker in its first 30 lines opts itself out. Narrow, but it's the one prose
  case the anchoring doesn't catch.

- **Tags `v0.3.0`–`v0.5.0` point at pre-rewrite commits.** They keep all 52
  original commits reachable on GitHub, so the clean four-commit history isn't
  actually the only history. Decide whether to repoint, delete, or keep them as
  an archive — and note there's no `v0.6.0` tag despite the version bump.

- **"Windows is coming soon"** now appears in both the README and
  `docs/install.md`. The daemon already has Windows paths (a Startup-folder
  `.vbs`, `windowsHide`), so the claim is plausible — but it's a promise with no
  date and no CI coverage behind it.

- **The site URL is hardcoded in four places.** `og:url`, `og:image`, the
  canonical link, and the OG card's own footer text all spell out
  `anteprojector.github.io/sidecar`. If a custom domain ever lands, they have to
  move together, and the one inside the PNG needs a regenerate.
