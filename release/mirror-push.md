# pdxdig -- mirror-push runbook (pdxdig#15, M5-002)

**Status:** BLOCKED on the R32 signed-release infra landing. No push
happens as part of this milestone; this document names the target,
the workflow a future operator runs, and the checks that gate it.

## 1. Mirror URL pattern

    https://pkgs.paideia-os/main/pdxdig/<version>/

For this release:

    https://pkgs.paideia-os/main/pdxdig/1.4.0/manifest.pdxsig
    https://pkgs.paideia-os/main/pdxdig/1.4.0/pdxdig.elf
    https://pkgs.paideia-os/main/pdxdig/1.4.0/caps.decl

Layout on the mirror mirrors `release/RELEASE-1.4.0.md` §5's own
distribution note:

    /pkgs/pdxdig-1.4.0/
        bin/pdxdig
        caps.decl
        manifest.pdxsig
    /bin/pdxdig -> /pkgs/pdxdig-1.4.0/bin/pdxdig

## 2. Push workflow (blocked on R32)

The push is the last step of the standard cut-a-release procedure
(`release/RELEASE-1.4.0.md` §4, Step 6), invoked only after the
compiled artifact is built (Step 3) and the manifest is dual-signed
(Step 5):

    paideia-release mirror-push \
        --repo    https://pkgs.paideia-os/main/ \
        --pkg     pdxdig \
        --version 1.4.0 \
        --files   build-out/pdxdig.elf \
                  caps.decl \
                  build-out/manifest.pdxsig

This requires:

- **R32 signed-release infra** -- the `pkgs.paideia-os` host and the
  `paideia-release` CLI's `mirror-push` subcommand. Neither exists as
  of this milestone (same posture every other R100 satellite's own
  `RELEASE-*.md` runbook records at S5).
- A `manifest.pdxsig` whose two signature blocks are real ML-DSA-65 +
  Ed25519 signatures, not the
  `SIGNATURE_PLACEHOLDER_PENDING_LIVE_SIGN` sentinel this repo ships
  today -- see `release/manifest.pdxsig.txt` §[signatures] and
  `design/02-development-environment.md` §1164 for the hardware-backed
  key custody the real sign pass draws from.

## 3. Verification steps once R32 lands

A future operator, after a real push, verifies the mirror copy before
trusting it:

1. **Fetch + hash-check.** Download
   `https://pkgs.paideia-os/main/pdxdig/1.4.0/manifest.pdxsig` and
   `pdxdig.elf`; recompute each artifact's BLAKE3-256 and compare
   against the manifest's `[artifacts.*]` table.
2. **Dual-signature verify.** Run both signature checks with
   AND-semantics (`design/02-development-environment.md` §1140):
   Ed25519 verify against `ed25519-key-id`'s known public key, and
   ML-DSA-65 verify against `ml-dsa-65-key-id`'s. Reject the package
   if either fails.
3. **Tag/commit cross-check.** Confirm `source-commit` in the pushed
   manifest matches the commit `git tag v1.4.0` points at in this
   repo, so the mirror cannot silently serve a different tree under
   the same version string.
4. **`pkg install --strict` dry-run.** Once a package-tool client
   exists, a `--strict` install against the pushed manifest should
   succeed cleanly -- today it would refuse (exit 4, cap denied per
   I4) against the placeholder-signed source form, which is the
   expected and desired refusal until Step 2 above actually runs.
5. **Cross-repo consistency.** Confirm the pushed `caps.decl` matches
   this repo's `caps.decl` byte-for-byte (no drift between what was
   signed and what is served).

Until R32 lands, steps 1-5 above are not executable against a real
endpoint; this section documents the check sequence for the operator
who performs the first real push.
