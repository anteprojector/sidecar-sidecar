# A `requires` version range in .sidecar

Status: design musing, 2026-07-28. Not committed to building this; captured
while the thinking was fresh. The newest-install-wins resolver (implemented)
is the degenerate case of this design; `requires` is the escape hatch we
deliberately deferred until a repo actually needs it.

## The problem it solves

The resolver now runs the newest of the global and project-local installs.
That fixes the everyday pain — update the global once, every repo gets it —
but it removes the only way a repo could hold itself on an old version. Our
fix policy is roll-forward ("if the global breaks we ship a new global"), so
*we* never need a cap. A repo that can't upgrade for its own reasons might.

## The design

Policy and mechanism split cleanly:

- **`.sidecar` declares policy**: a committed `requires` key holding a
  version range. Committed means every machine and teammate enforces the
  same policy — unlike a devDependency pin, which only binds machines that
  happen to have run `npm install`.
- **The node dependency provides mechanism**: `devDependencies` is how a
  repo *supplies* a version that satisfies its range. Its other job
  (postinstall self-registration on fresh clones) is unchanged.

The resolver becomes: **run the highest installed version that satisfies
`requires`**. No `requires` means `*`, which is exactly today's newest-wins.
A capped repo sets e.g. `requires = ">=1.2 <1.4"` and keeps a local install
inside the range; the too-new global defers to it.

## Decisions already made

- **Key name**: not `version` — `.sidecar` already uses that for the config
  schema version. `requires` reads well.
- **Grammar**: a conjunction of plain comparators (`>=X`, `<=X`, `<X`,
  exact). Not full npm semver — hand-rolling `^`/`~`/`||`/x-ranges invites
  divergence from npm semantics, and bundling the `semver` package fattens
  the committed dist for grammar nobody needs. Reject anything else loudly.
- **Fail closed, actionably**: when no installed sidecar satisfies the
  range, commands error with the fix spelled out ("no installed sidecar
  satisfies >=1.2 <1.4 — npm i -D sidecarsync@1.3"). Exception: `status`
  still renders and shows the mismatch as a warning; it's the diagnostic
  tool.
- **Daemon behavior**: a repo whose range excludes the daemon syncs via a
  satisfying local CLI (the pinned-CLI path already exists). If the daemon
  is *below* the range and autoupdate is on, that's a reason to pull the
  update now rather than wait for the daily check; if nothing satisfies,
  skip the repo and surface it through `sidecar health` — a capped repo
  quietly failing is exactly the one-machine signal health exists for.
- **No auto-stamp at init**: a fresh repo has no version requirement, and
  stamping the current version would create fake floors everywhere. It's a
  declaration you add when the repo starts depending on something.

## Known sharp edges

- **A max with no local install is a time bomb**: the global marches
  forward, one day crosses the cap, and every command in that repo fails.
  `status`/`health` should warn as soon as the global exceeds the max and a
  local copy is the only thing keeping the repo alive — that repo has
  silently opted out of fixes.
- **Bootstrap gap**: old CLIs ignore unknown `.sidecar` keys (readConfig
  plucks known keys only), so adding `requires` is compatible — but a
  machine running an old global won't *enforce* it until it updates once
  past the release that ships this. The floor doesn't protect genuinely
  stale machines.
- **Keeps the skew machinery**: a capped repo means a newer daemon drives an
  older CLI, which is what the env-var compat contract (`SIDECAR_SYNC_SOFT`,
  `SIDECAR_SYNC_LOCAL`) exists for. That contract has to survive anyway for
  cross-machine skew; `requires` just adds a deliberate consumer.
