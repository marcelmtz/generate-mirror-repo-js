# Proposal: a six-month grace window for the previous release line

**Status:** draft for discussion
**Date:** 2026-08-31

## Problem

Today a Mage-OS release line stops the moment the next one ships. Merchants and
agencies get no overlap in which to plan an upgrade.

Tracing `extra.magento_version` across `resource/history/mage-os/`:

| Mage-OS | Upstream base | Released |
| --- | --- | --- |
| 2.2.2 | 2.4.8-p4 | 2026-05-12 |
| **2.3.0** | **2.4.8-p5** | **2026-05-17 — last 2.x ever** |
| 3.0.0 | 2.4.9 | 2026-06-15 |
| 3.1.0 | 2.4.9 | 2026-07-13 |
| 3.3.0 | 2.4.9 | 2026-08-10 |

Magento 2.4.9 reached GA on 2026-05-12; Mage-OS 3.0.0 followed 34 days later.
The 2.x line has had **zero** releases since.

Meanwhile **Adobe supports the 2.4.8 base until 2028-05-31** and is still
publishing `-pN` patches for it. The security work is already done upstream —
we simply stopped forwarding it.

The same intent is encoded in `mage-os/github-actions` at
`supported-version/src/versions/mage-os/individual.json`, where each version's
`eol` is set to the *next version's release date*.

```mermaid
gantt
    title Mage-OS release lines against upstream support windows
    dateFormat YYYY-MM-DD
    axisFormat %b %Y

    section Upstream (Adobe)
    Magento 2.4.8 patched until May 2028   :done, u8, 2025-04-08, 2028-05-31
    Magento 2.4.9 patched until May 2029   :done, u9, 2026-05-12, 2029-05-31

    section Mage-OS today
    2.x stops dead at 3.0.0 GA             :crit, m2, 2026-01-06, 2026-05-17
    3.x current line                       :active, m3, 2026-06-15, 2027-06-15

    section With a grace window
    4.x next line (illustrative)           :active, m4, 2027-06-15, 2028-06-15
    3.x grace, 6 months past 4.0 GA        :g3, 2027-06-15, 2027-12-15
```

## Proposal

> Each Mage-OS major line continues to receive security and critical-fix
> releases for six months after the following major reaches GA. During that
> window we track upstream patch releases for the base Magento version.

The first line to get a grace window is **3.x, when 4.0.0 ships**. 2.x is not
retrofitted (see Recommendation).

The grace line is **the last 3.x release available when 4.0.0 ships**. The
examples below use 3.6.0 for it; the real number is whatever 3.x is at by then.
Merchants on earlier 3.x releases get fixes by moving to the latest 3.x — a
step within the same line, on the same Magento 2.4.9 base, which a `^3.0`
constraint picks up automatically. That mirrors Adobe, which supports the latest
patch of each line rather than every release in it.

Scope rules:

- **Security and critical regressions only.** No feature backports.
- **Patch-level versions only** — `3.6.1`, `3.6.2`. Never a new minor on a line
  in its grace window; a minor signals features.
- **Event-driven, not scheduled.** We build the old line only when Adobe ships a
  patch for its base. No upstream drop, no release.

## This is an overlap window, not an LTS

The distinction carries most of the cost. A true LTS runs years past upstream
and eventually means originating our own security patches — that needs funding
or a separate team, which is how Ubuntu and Debian do it.

A six-month grace never approaches that. The old base stays inside Adobe's
support window for the whole period, so every grace release is **forwarding
work, not authoring work**. 3.x is built on Magento 2.4.9, which Adobe supports
until 2029-05-31, so a six-month window after 4.0.0 closes long before that.

Adobe's patch rhythm is close to bi-monthly (`2.4.8` Apr 2025 → `p1` Jun →
`p2` Aug → `p3` Oct → `p4` Mar 2026 → `p5` May), so a six-month window catches
two or three drops: two or three extra builds per cycle, against twelve to
fifteen for a real LTS.

## Mechanics

`main` will have moved to 4.x, so the grace line cannot be built from it, nor
from a tracking branch: for 2.x, `2.3.0...2.4-develop` in `mageos-magento2` was
**diverged, 10,272 commits ahead**. The grace line branches from the commit the
last 3.x release was built from, and each security fix is ported onto it.

```mermaid
gitGraph
    commit id: "3.5.0 source"
    commit id: "3.6.0 source" tag: "built as 3.6.0"
    branch release/3.x
    checkout main
    commit id: "4.x work"
    commit id: "4.0.0 source" tag: "built as 4.0.0"
    checkout release/3.x
    commit id: "port Adobe patch"
    commit id: "3.6.1 source" tag: "built as 3.6.1"
```

1. **At 4.0.0 GA, point `release/3.x` at the commit 3.6.0 was built from** —
   the parent of the CI "Release 3.6.0" commit the tag sits on, which only
   rewrites versions.
   The existing `release/3.x` in `mageos-magento2` is the branch used to prepare
   3.0: its last commit is from 2026-05-18, it is 59 commits behind 3.4.0, and
   it has no commits of its own, so resetting it loses nothing.
2. **Port Adobe's isolated patch onto it.** Adobe publishes one per upstream
   line; `bin/apply-security-patch.js` from #356 rewrites its `vendor/` paths
   to source-tree paths and applies it, one review branch per line.
3. **Build with a release refs file.**

## Required changes

Only two repositories need edits. The ~25 `mageos-*` build repos need
**branches, not code**.

### 1. `mage-os/generate-mirror-repo-js` (this repo) — included in this PR

- [x] `src/make/mageos-release.js`, `src/release-build-tools.js` — a new
      release now honours each repository's and metapackage's `fromTag`, which
      until now only the history rebuild checked. Without this, a release on
      an older line built with a later config picks up repositories and
      metapackages that start after it — for example, a 2.x release would get
      `magento-zf-captcha`, `magento-zf-soap` and the minimal edition, which
      start at 3.0.0. Releases on the current line are unaffected.
- [x] `.github/workflows/build-mageos-release.yml` — new optional
      `release_refs_file` input.
- [x] `.github/workflows/deploy.yml` — threads it through as
      `--releaseRefsFile`. Empty by default, so existing runs are unchanged.
- [x] `src/build-config/mage-os-release-refs/README.md` — documents the
      convention. No version file is included; adding one would change how that
      version builds.

`--releaseRefsFile` parsing (#353) and per-repository keys in the refs file
(#354) were already fixed on `main`.

A release creates its tag in every repository from that repository's ref, so a
refs file only needs a branch where there are patches. Everything else builds
from the outgoing line's last tag, which every repository already has:

```js
// src/build-config/mage-os-release-refs/3.6.1.js
module.exports = {'*': '3.6.0', 'magento2': 'release/3.x'};
```

**Minimal edition.** It is built by the same release, as the
`product-minimal-edition` and `project-minimal-edition` metapackages, so it
follows the same grace window: a 3.x security release after 4.0 includes it. The supported-version data in `mage-os/github-actions` has no
minimal-edition entries at all, so it is not in the CI matrix today either.

No change is needed in `.github/workflows/push-release-tag.yml`: it pushes tag
objects from the build's working copies, so a tag created on `release/N.x`
already points at the right commit.

### 2. `mage-os/github-actions` — a data change, not code

- [ ] `supported-version/src/versions/mage-os/individual.json` — today each
      version's `eol` is the next version's release date. That stays right for
      3.0.0 to 3.5.0, since only the latest 3.x is supported. Two changes:
      - the last 3.x release (3.6.0) gets `eol` = **4.0.0's release date plus
        six months**;
      - each grace release (3.6.1, 3.6.2, …) gets its own entry with the same
        `eol`.
- [ ] Add the minimal edition, which has no entries today.

`getCurrentlySupportedVersions` already filters on `release`/`eol`, so no new
mechanism is required — only the dates change.

### 3. The `mageos-*` build repositories — no code changes

- [ ] Branch `release/3.x` **lazily** — only in repos that receive a patch.
      The rest build from their `3.6.0` tag through the refs file's `*` key.
- [ ] In `mageos-magento2`, reset the existing, stale `release/3.x` at 4.0.0
      GA (see Mechanics).

### 4. Release skills — follow-up, deliberately not in this PR

`prep-release-prs` merges `release/N.x` **into** the default branch, and
`release/3.x` is exactly the name it would pick up. Under this
policy a grace branch must not be merged into trunk while it is live, so that
skill needs a guard before the first grace window opens — it is the one that
could actively cause harm. `audit-release-branches` would want to distinguish a
pre-release staging branch from a live grace branch, and `merge-upstream-tracking`
targets default branches only.

None of these bite until a grace branch exists, and they encode workflow that
depends on the policy being agreed first, so they are left for a follow-up.

## Open question: major cadence

Mage-OS majors currently arrive faster than six months — 2.0.0 on 2026-01-06,
3.0.0 on 2026-06-15: about **5.3 months**. At that pace a six-month grace means
two lines are live permanently, occasionally three, and the window never closes.

Options:

1. **Tie majors to upstream minors** (roughly annual: 2.4.7 Apr 2024 → 2.4.8
   Apr 2025 → 2.4.9 May 2026). Two lines for half of each year, closing cleanly.
   Recommended.
2. **Shorten the grace to three or four months** — still catches two upstream
   drops, still closes.
3. **Accept permanent two-line operation** and staff for it.

1.0.0 was only released in November 2025, so the fast major cadence may be
early-project churn rather than steady state.

## Recommendation

Adopt the policy **from the 3.x → 4.x transition**, and do not retrofit 2.x:

- 2.x ended in May 2026 and 3.x has had five releases since. Most people who
  were going to move have moved.
- Retrofitting would not be free. Adobe has published isolated patches for
  2.x's base (2.4.8-p5) in July, August and September 2026; #356's demo ported
  all of them, with one hunk resolved by hand. Supporting 2.x now means
  shipping that backlog.
- Starting with 3.x means the branch is reset at the right moment rather than
  reconstructed after the fact.

For current 2.x holdouts the honest message is that 2.x remains installable
indefinitely from repo.mage-os.org and the supported path is 3.x — with a
commitment that this is the last time a line ends without warning.

## Effort

- **One-time:** roughly a week. The refs-file wiring is trivial; the branch and
  tag plumbing is the real work.
- **Recurring:** two or three extra builds per major cycle, only when upstream
  ships a patch.
- Subsequent majors repeat the same steps: at 5.0.0, the last 4.x release
  becomes the grace line.

---

*Sources: `resource/history/mage-os/` and `resource/history/magento/` in this
repository; branch and tag state on `mage-os/mageos-magento2`,
`mage-os/mirror-magento2` and `mage-os/github-actions`; Adobe Commerce lifecycle
policy, checked 2026-08-31.*
