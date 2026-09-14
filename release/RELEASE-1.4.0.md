# pdxdig v1.4.0 -- release note + mirror-push workflow (Wave GGG drain)

**Repo:** github.com/paideia-os/pdxdig
**Wave:** R100 user-space networking tools
(`design/networking/r100-user-tools-plan.md` §6 + §13.5, in the
paideia-os monorepo).
**Version at this release:** 1.4.0 (Wave GGG drain, closing
pdxdig#14, #15 -- follows the unsigned v1.1.0, v1.2.0, v1.3.0 source
tags).
**Upstream policy:** `design/tooling/plan.md` §6.3 (paideia-os) --
package repository layout; `design/02-development-environment.md`
§1140 + §1164 -- hybrid Ed25519 + ML-DSA-65 signing, release-line key
custody; `design/networking/r100-user-tools-plan.md` §13.5 --
pdxdig's release-closer scope.

This is a **docs/release-only** bump: no file under `src/` or
`tests/` changes in this landing. It follows the same
`release/RELEASE-1.*.md` runbook shape as v1.3.0 -- see that document
for the worked template this one follows.

The actual `git tag v1.4.0` + `git push` is a **manual step main
performs separately from this milestone** -- see §4.

---

## 1. What v1.4.0 ships

- **`doc/pdxdig.pdxdoc` (pdxdig#14, M5-001)** -- `pdxdoc-source v0.1`
  documentation source, wired into `manifest.pdxproj`'s previously
  empty `docs:` list.
- **`release/manifest.pdxsig.txt`** -- bumped to package-version
  1.4.0 / `source-tag: v1.4.0`; artifact list extends to cover
  `doc/pdxdig.pdxdoc`, `release/RELEASE-1.4.0.md`, and
  `release/mirror-push.md`. Both `[signatures]` blocks remain the
  documented `SIGNATURE_PLACEHOLDER_PENDING_LIVE_SIGN` sentinel.
- **`release/mirror-push.md` (pdxdig#15, M5-002)** -- documents the
  `pkgs.paideia-os` mirror URL pattern, push workflow, and
  verification steps once R32 signed-release infra lands.
- **`CHANGELOG.md`** -- new `[1.4.0]` stanza.

What v1.4.0 does NOT change: everything under `src/` and `tests/`,
`link.ld`, `tools/build.sh`, `LICENSE`, `caps.decl`. `PDX_TOOL_VERSION`
in `src/tool_ident.pdx` stays `"1.3.0\0"` -- reconciling it with the
tag is left to the next source-bearing wave, since this landing is
IMPL-ONLY docs/tags per the batch directive it was cut under.

## 2. What v1.4.0 explicitly does NOT ship (deferred)

- **Real dual-sign pass.** See §3 below; this milestone lands the
  *source form* updates only.
- **Mirror push to `pkgs.paideia-os/main/pdxdig/1.4.0/`** -- endpoint
  does not exist as of this milestone. `release/mirror-push.md`
  documents the target layout and is BLOCKED on the R32 signed-release
  infra landing.
- Every item v1.3.0's own §2 already deferred (real UDP query
  round-trip, `--server=` consumption, `--timeout-ms=` semantics,
  real `libpdx-audit` fingerprint write) is unchanged and still
  deferred -- this release touches none of that surface.

## 3. Substrate readiness (blocking the actual signed release)

Unchanged from v1.3.0 §3 (S1..S5): paideia-as toolchain floor,
`paideia-pq-sign::sign_release_artifact` reachability, live
release-line seed key custody (§1164), and the non-existent
`pkgs.paideia-os` mirror endpoint (S5) all still gate the real
dual-sign + push pass. Nothing in this landing changes that posture;
`release/mirror-push.md` names the R32 signed-release infra tracking
item explicitly as the blocker for S5's push step.

---

## 4. Cut-a-release procedure

**Pre-flight.**

    git fetch origin
    git switch main
    git pull --ff-only
    git status                    # MUST be clean
    gh issue list --state open --repo paideia-os/pdxdig
                                  # MUST be empty of the Wave GGG cohort

**Step 1 -- Version bump + CHANGELOG close.** Already done at this
milestone -- `CHANGELOG.md`'s `## [1.4.0]` entry is the one the tag
points at, `manifest.pdxproj`'s `version = 1.4.0` is set.

**Step 2 -- Tag.** (Manual, main-performed; NOT run as part of this
milestone.)

    git tag -a v1.4.0 -m "pdxdig v1.4.0 -- R100 Wave GGG drain"
    git push origin v1.4.0

**Steps 3-7** mirror v1.3.0's own §4 Steps 3-7 exactly (build, fill
manifest, dual-sign, mirror-push, GitHub release) with `1.3.0`
replaced by `1.4.0` throughout. See that document for the full
command listing; not repeated here to avoid drift between the two
runbooks.
