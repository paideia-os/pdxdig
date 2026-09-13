# pdxdig v1.3.0 -- release note + mirror-push workflow (Wave GG drain)

**Repo:** github.com/paideia-os/pdxdig
**Wave:** R100 user-space networking tools
(`design/networking/r100-user-tools-plan.md` §6 + §13.5, in the
paideia-os monorepo).
**Version at this release:** 1.3.0 (Wave GG drain, closing pdxdig#4,
#5, #7, #9, #18 -- follows the unsigned v1.1.0 and v1.2.0 source
tags).
**Upstream policy:** `design/tooling/plan.md` §6.3 (paideia-os) --
package repository layout; `design/02-development-environment.md`
§1140 + §1164 -- hybrid Ed25519 + ML-DSA-65 signing, release-line key
custody; `design/networking/r100-user-tools-plan.md` §13.5 --
pdxdig's release-closer scope.

This document is both the **release note** for what v1.3.0 ships and
the operator runbook for cutting the signed release and pushing it to
the paideia-os package mirror at
`https://pkgs.paideia-os/main/pdxdig/1.3.0/`. The workflow mirrors
`pdxsock`'s, `mkfs.pdxfs`'s, and `libpdx-audit`'s own
`release/RELEASE-1.*.md` runbooks -- see any of the three for the
worked template this one follows.

The actual `git tag v1.3.0` + `git push` is a **manual step main
performs separately from this milestone** -- see §4. This document
describes exactly what that tag would contain so a future operator
can cut it without re-deriving scope from the CHANGELOG and five
milestone flags.

---

## 1. What v1.3.0 ships

- **Real `--server=` IPv4 override (pdxdig#5, M2-002)** --
  `ArgvParse::argv_parse_ipv4` in `src/argv_parse.pdx`; `Main::_start`
  stores the parsed value in `dig_server_override_ip` /
  `dig_server_override_set`, or refuses (exit 2) on a malformed value.
- **M2-001 UDP-query extern scaffold (pdxdig#4)** -- `src/dig_query.pdx`
  documents the real `net_resolve` contract and ships a WEAK-STUB
  `dig_query_resolve` returning the `0xDEADBEEF` sentinel. Not yet
  called from `_start`; BLOCKED on R100-PREP-002 / §12.1.
- **M3-001 libpdx-audit STUB wire (pdxdig#7)** -- `src/audit_wire.pdx`'s
  `pdxdig_audit_log_query`, called once per accepted query from
  `_start`; STUB body always returns `AFS_OK` (0).
- **M4-001 happy-path A-record smoke (pdxdig#9)** --
  `tests/dig_a_smoke.pdx` mocks an `example.com` -> `93.184.216.34`
  resolver reply and asserts the extracted answer + fd-2 fingerprint.
- **`manifest.pdxproj`** -- `version` bump `1.2.0 -> 1.3.0`; new
  `sources:` (`dig_query.pdx`, `audit_wire.pdx`) and `tests:`
  (`dig_a_smoke.pdx`) entries.
- **`src/tool_ident.pdx`** -- `PDX_TOOL_VERSION` bumped to `"1.3.0\0"`.
- **`release/manifest.pdxsig.txt`** (new, this repo) -- dual-sign
  source form, pdxsock/mkfs.pdxfs-template shape. Every `<BLAKE3-*>`
  / `<...-KID-*>` / `<...-SIG-*>` slot is a documented placeholder the
  release tool fills in at tag time (see §3).
- **`release/RELEASE-1.3.0.md`** (new, this repo) -- this document.
- **`CHANGELOG.md`** -- new `[1.3.0]` stanza; **`README.md`** --
  documents the real `--server=` parse + the M2-001/M3-001 blocker
  status.

What v1.3.0 does NOT change: `src/dns_walker.pdx`, `link.ld`,
`tools/build.sh`, `LICENSE`, `caps.decl`'s cap-kind set. Every
existing syscall path (sysnos 0/1/60/115) is byte-for-byte the
v1.2.0 body outside the new `--server=`/AuditWire call sites this
release adds.

## 2. What v1.3.0 explicitly does NOT ship (deferred)

- **Real UDP query round-trip** -- pdxdig#4 / M2-001, BLOCKED on
  R100-PREP-002 (§12.1 kernel-side SOCK_DGRAM extension). The no-flag
  argv path still exits 3 with `pdxdig: udp stub\n`.
- **`--server=` actually changing which resolver is queried** -- no
  real resolver-selection code exists yet; the override is parsed and
  stored, not consumed (M2-001-b follow-up).
- **`--timeout-ms=` semantics** -- recognized, no-op (M2-004 follow-up).
- **Real libpdx-audit fingerprint write** -- pdxdig#7 / M3-001-b,
  gated on the `libpdx-audit` dep actually landing in `deps:` without
  breaking this repo's flat single-ELF `--fatal-warnings` link.
- **QEMU-substrate smokes beyond the fixture-harness pattern** --
  runtime round-trip witnessing is the paideia-os smoke-runner's job
  (matches pdxsock's own M4-001..M4-003 precedent).
- **Dual-signed `manifest.pdxsig` binary** -- see §3 below; this
  milestone lands the *source form* of the manifest but does not
  execute the sign pass.
- **Mirror push to `pkgs.paideia-os/main/pdxdig/1.3.0/`** -- endpoint
  does not exist as of this milestone; §5 below documents the layout
  a future operator pushes.

## 3. Substrate readiness (blocking the actual signed release)

**S1 -- paideia-as toolchain >= 0.36.0 reachable.** `tools/build.sh`
already asserts this floor and refuses fast on any older toolchain.

**S2 -- `paideia-pq-sign::sign_release_artifact` reachable.** Same
release-time dual-sign entrypoint every other satellite's runbook
depends on.

**S3 -- live release-line seed keys.** Out-of-repo, hardware-backed
custody per `design/02-development-environment.md` §1164; the
`[signatures]` block of `release/manifest.pdxsig.txt` carries the
`SIGNATURE_PLACEHOLDER_PENDING_LIVE_SIGN` sentinel in every slot.

**S4 -- `doc` M2 reachable.** No `.pdxdoc` source exists in this repo
yet (`manifest.pdxproj` `docs: []`); user documentation lands at a
future M5-001.

**S5 -- `pkgs.paideia-os` mirror endpoint reachable.** Same
non-existent-as-of-R100-close status every earlier satellite's own
runbook documents.

---

## 4. Cut-a-release procedure

**Pre-flight.**

    git fetch origin
    git switch main
    git pull --ff-only
    git status                    # MUST be clean
    gh issue list --state open --repo paideia-os/pdxdig
                                  # MUST be empty of the Wave GG cohort

**Step 1 -- Version bump + CHANGELOG close.** Already done at this
milestone -- `CHANGELOG.md`'s `## [1.3.0]` entry is the one the tag
points at, `manifest.pdxproj`'s `version = 1.3.0` is set.

**Step 2 -- Tag.** (Manual, main-performed; NOT run as part of this
milestone.)

    git tag -a v1.3.0 -m "pdxdig v1.3.0 -- R100 Wave GG drain"
    git push origin v1.3.0

**Step 3 -- Build the compiled artifact set.**

    bash tools/build.sh
    # emits build-out/pdxdig.elf + build-out/pdxdig.bin

**Step 4 -- Recompute the manifest.**

    paideia-release fill-manifest \
        --source release/manifest.pdxsig.txt \
        --tree   . \
        --tag    v1.3.0 \
        --output build-out/manifest.pdxsig.filled.txt

**Step 5 -- Dual-sign.**

    paideia-release sign \
        --manifest build-out/manifest.pdxsig.filled.txt \
        --key-ed25519  release-line-ed25519.sk \
        --key-ml-dsa65 release-line-ml-dsa-65.sk \
        --output   build-out/manifest.pdxsig

**Step 6 -- Mirror push.** Out of scope at THIS milestone (no
`pkgs.paideia-os` endpoint exists yet -- see S5).

    paideia-release mirror-push \
        --repo   https://pkgs.paideia-os/main/ \
        --pkg    pdxdig \
        --version 1.3.0 \
        --files  build-out/pdxdig.elf \
                 caps.decl \
                 build-out/manifest.pdxsig

**Step 7 -- GitHub release.**

    gh release create v1.3.0 \
        --title "pdxdig v1.3.0" \
        --notes-file CHANGELOG.md \
        build-out/manifest.pdxsig \
        build-out/pdxdig.elf \
        caps.decl

---

## 5. Distribution -- what would be pushed to `pkgs.paideia-os`

**Scope note:** this section documents the mirror-push target layout
for a future operator to execute against a real `pkgs.paideia-os`
endpoint (S5 above -- does not exist yet). No actual push happens as
part of THIS milestone, and no `pkgs.paideia-os` artifacts are
created by this landing.

Expected mirror layout after a real Step 6 push:

    /pkgs/pdxdig-1.3.0/
        bin/pdxdig                 # the linked, runnable ELF (Step 3)
        caps.decl                  # this repo's capability declaration
        manifest.pdxsig            # dual-signed binary manifest (Step 5)
    /bin/pdxdig -> /pkgs/pdxdig-1.3.0/bin/pdxdig
                                    # the package tool's own symlink
