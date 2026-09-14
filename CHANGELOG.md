# Changelog

All notable changes to `pdxdig` are recorded here. Format: keep-a-
changelog-style, semver-ordered, newest first.

<!--
Version discipline:
  v1.0.x -- reserved for the original M5 dual-signed 1.0-shape plan.
             No v1.0.0 tag was ever cut.
  v1.1.0 -- Wave L drain: source scaffold + first-real-body tool
             + tests/*.pdx witnesses (pdxdig#10, #11, #12, #13, #17).
  v1.2.0 -- Wave W drain: argv-surface parser + --dry-run/--type=
             first-runnable branches + M1-001/M3-002 honest witnesses
             (pdxdig#1, #2, #3, #6, #8).
  v1.3.0 -- Wave GG drain: real --server= IPv4 override, M2-001
             net_resolve extern scaffold (WEAK stub), M3-001
             libpdx-audit STUB wire, M4-001 happy-path A-record
             smoke, release closer (pdxdig#4, #5, #7, #9, #18).
  v1.4.0 -- Wave GGG drain: docs/release-only bump -- dual-signed
             manifest.pdxsig + .pdxdoc (M5-001) + mirror-push runbook
             (M5-002). No src/ or tests/ file changes (pdxdig#14, #15).
-->

## [Unreleased]

## [1.5.0] -- 2026-09-14 -- Wave mu-03: real DNS-over-TLS path (pdxdig#8)

### Added
- **`src/dot_wire.pdx` (Module DotWire), gated behind a new `--dot=1`
  argv flag.** A REAL DNS-over-TLS-shaped query path, distinct from
  (and NOT blocked by) `src/dig_query.pdx`'s fully-WEAK UDP scaffold:
  a real 12-byte DNS header + real RFC-1035 label-encoded QNAME +
  QTYPE=A/QCLASS=IN, length-prefixed per RFC 7858/7766, sent over a
  REAL `sys_socket`+`sys_connect` TCP connection to `<resolver>:853`,
  through a `net_tls_wrap`-shaped WEAK-passthrough call
  (`dot_wire_net_tls_wrap` -- libpdx-net not yet linkable from a
  satellite repo, its own `net_tls_wrap` itself still scaffold-only),
  then a REAL `sys_send`/`sys_recv` round trip (query travels in
  plaintext at this landing). `dot_wire_query` returns the real
  received byte count on success or one of five distinct negative
  status codes (SOCKET_FAIL/CONNECT_FAIL/SEND_FAIL/RECV_FAIL/
  QNAME_TOO_LONG). An oversized qname argument is rejected cleanly
  (bounds-checked write cursor) rather than overflowing the query
  buffer.
- **`src/main.pdx`: `--dot=1` argv recognition + dispatch.** When
  present, resolves the target IP (the existing `--server=` override
  if set, else a default `8.8.8.8`) and calls `dot_wire_query`,
  printing `pdxdig: dot bytes=<n>` (fd 1, exit 0) on success or
  `pdxdig: dot fail rc=<n>` (fd 2, new exit code 6) on failure --
  bypassing the UDP-stub/--dry-run/--type= terminal dispatch entirely.
  New shared helper `pdxdig_print_decimal(fd, value)`.
- `caps.decl`: new optional `KIND_TCP_SOCKET` row for the real
  TCP I/O `--dot=1` performs.

### Known gaps
- No encryption (no crypto intrinsics exist in the paideia-as stdlib
  yet); the DoT response is not parsed into a `DnsQueryRecord@0.1`
  answer (raw byte count only); single-shot `sys_send`/`sys_recv`
  (no loop to reassemble a split TCP read).

## [1.4.0] -- 2026-09-13 -- Wave GGG drain (Closes #14. Closes #15.)

Docs/release-only bump on top of v1.3.0 -- no source or test file
under `src/` or `tests/` changes in this landing.

### Added
- **`doc/pdxdig.pdxdoc` (pdxdig#14, M5-001).** `pdxdoc-source v0.1`
  documentation source (NAME / SYNOPSIS / DESCRIPTION / OPTIONS /
  EXIT-CODES / RECORD / AUDIT / LIMITATIONS / SEE-ALSO), wired into
  `manifest.pdxproj`'s previously-empty `docs:` list.
- **`release/manifest.pdxsig.txt`** bumped to package-version 1.4.0 /
  `source-tag: v1.4.0`; artifact list extends to cover the new
  `doc/pdxdig.pdxdoc` entry. Both `[signatures]` blocks remain the
  documented `SIGNATURE_PLACEHOLDER_PENDING_LIVE_SIGN` sentinel --
  live release-line key material still lives outside every repo per
  `design/02-development-environment.md` §1164.
- **`release/RELEASE-1.4.0.md`** (new) -- release note + cut-a-release
  runbook, following the v1.3.0 template.
- **`release/mirror-push.md` (pdxdig#15, M5-002).** Documents the
  `pkgs.paideia-os` mirror URL pattern, the push workflow, and the
  verification steps a future operator runs once the R32
  signed-release infra lands. No push happens as part of this
  milestone -- the `pkgs.paideia-os` endpoint does not exist yet.

### Unchanged
- Every file under `src/` and `tests/` is byte-for-byte the v1.3.0
  body. `PDX_TOOL_VERSION` in `src/tool_ident.pdx` stays `"1.3.0\0"`
  in this landing (a source edit is out of scope for a docs-only
  release closer); the next source-bearing wave reconciles it with
  the `v1.4.0` tag.

## [1.3.0] -- 2026-09-13 -- Wave GG drain

### Added
- **`--server=` real IPv4 override (pdxdig#5, M2-002).** New
  `ArgvParse::argv_parse_ipv4(str_ptr) -> u64` leaf in
  `src/argv_parse.pdx`: byte-scans a dotted-quad, returning a packed
  MSB-first `u32`-in-`u64` on a well-formed 4-octet value (0..255 per
  octet, exactly 3 `.` separators) or the all-ones sentinel
  `0xFFFFFFFFFFFFFFFF` on any malformation (empty octet, octet > 255,
  wrong octet count, non-digit byte). `Main::_start`'s `--server=`
  arm now calls this helper on the bytes past the prefix and either
  stores the packed value into the new module-level
  `Main::dig_server_override_ip` (+ sets `dig_server_override_set =
  1`) on success, or folds a malformed value into the same
  usage-refusal bucket (exit 2) an unknown `-flag` already uses. No
  real resolver-selection code exists yet to consume the override
  (that arrives with M2-001) -- this is the argv-surface half of the
  contract, landed independently per the issue's own scoping.

- **M2-001 UDP-query extern scaffold (pdxdig#4).** New
  `src/dig_query.pdx` (module `DigQuery`) documents the real
  `net_resolve(host_ptr, host_len, qtype, out_addr) -> u64` extern
  contract libpdx-net will expose once R100-PREP-002 / §12.1
  (kernel-side SOCK_DGRAM extension) unblocks it, and provides
  `dig_query_resolve` -- a WEAK STUB with the IDENTICAL 4-arg ABI
  that ignores every input, writes the LE sentinel `0xDEADBEEF` into
  `[out_addr]` (guarded against a null out-param), and returns
  `0x00000000DEADBEEF` in `rax`. No bare `call net_resolve` is
  emitted anywhere in this repo -- `tools/build.sh` links every
  `src/*.o` into one flat ELF with `--fatal-warnings`, so an
  unresolved UND symbol would break every build, not just this call
  site, until libpdx-net actually lands in `deps:` (see file header
  for the full rationale, matching `paideia-os/mv`'s `ElevateGate`
  and `pdxsock`'s `AuditWire` precedent). NOT YET called from
  `Main::_start` -- pdxdig#4 is scoped as "scaffold the wire".

- **M3-001 libpdx-audit STUB wire (pdxdig#7).** New
  `src/audit_wire.pdx` (module `AuditWire`) with
  `pdxdig_audit_log_query(qname_ptr, qtype, result_code) -> u64`,
  called from `Main::_start` once per argv-accepted invocation
  immediately after the `sys_semantic_send` emission ("Step 5b").
  Real body (once `libpdx-audit` lands in `deps:`) will format a
  compact query-name/qtype/rcode fingerprint line and call
  `AuditFileSink::audit_file_append("/system/audit/pdxdig.log", ...)`
  -- both documented verbatim in the file header. STUB body is
  `xor rax, rax; ret` (returns `AFS_OK = 0` unconditionally), same
  posture as pdxsock's own M3-002 `AuditWire` landing.

- **M4-001 happy-path A-record smoke (pdxdig#9).** New
  `tests/dig_a_smoke.pdx` (module `DigASmoke`): populates a 256-byte
  `.bss` fixture's answer-RDATA slot with the `example.com` A-record
  bytes (93.184.216.34), calls a test-local WEAK stub shaped exactly
  like the real `net_resolve` extern (`m9_net_resolve_stub`) to copy
  those bytes into an out-param, asserts the four resolved bytes
  match, and emits `pdxdig a-record ok 93.184.216.34\n` (33 wire
  bytes) on fd 2 + exit 0 on match, or `pdxdig a-record FAIL\n` (21
  wire bytes) + exit 1 on any mismatch.

### Changed
- `manifest.pdxproj` `version` bumped `1.2.0 -> 1.3.0`; `sources:`
  gains `src/dig_query.pdx` + `src/audit_wire.pdx`; `tests:` gains
  `tests/dig_a_smoke.pdx`.
- `src/tool_ident.pdx` `PDX_TOOL_VERSION` bumped `"1.2.0\0" ->
  "1.3.0\0"`.
- `deps.list` documents the `libpdx-audit` deferral alongside the
  existing `libpdx-net` one.
- `README.md` documents the real `--server=` parse + the M2-001/M3-001
  blocker status.

### Release (pdxdig#18, v1.1-C-shaped closer)
- `release/manifest.pdxsig.txt` (new) -- dual-sign source form,
  mkfs.pdxfs/pdxsock-template shape. Every `<BLAKE3-*>` /
  `<...-KID-*>` / `<...-SIG-*>` slot is a documented placeholder the
  release tool fills in at tag time.
- `release/RELEASE-1.3.0.md` (new) -- release note + mirror-push
  runbook, same template pdxsock's `RELEASE-1.2.0.md` establishes.
- `git tag v1.3.0` (next available semver after the already-tagged
  v1.1.0 / v1.2.0 -- the issue's own v1.1.0 suggestion predates those
  tags).

## [1.2.0] -- 2026-09-13 -- Wave W drain

### Added
- **argv-surface parser (pdxdig#2, M1-002).** New module
  `src/argv_parse.pdx` (module `ArgvParse`) with two pure-leaf
  SysV helpers -- `argv_parse_prefix(arg, prefix, plen)` for
  `--type=` / `--server=` / `--timeout-ms=` prefix detection and
  `argv_parse_eq_len(arg, wanted, wlen)` for the `--dry-run` exact
  gate (rejects `--dry-run-extra` via tail-NUL check). Main::_start
  is rewritten as a proper argv walk (register plan: r12=argc,
  r13=argv, r14=qname_ptr [last positional wins], r15=hash accum,
  rbx=flag bit-set [bit0=dry_run, bit1=unsupp_qtype, bit2=unknown
  flag], rbp=loop index). Unknown `-flag` -> usage exit 2; no
  positional -> usage exit 2; recognized-but-unhandled --server=
  and --timeout-ms= flow through as no-ops (real bindings arrive
  with M2-002 and a future M2-004 landing).

- **--dry-run first-runnable (pdxdig#3, M1-003).** The --dry-run
  path marshals the same DnsQueryRecord@0.1 with result_code =
  DRY_RUN (5), emits via `sys_semantic_send` (SC+ 115, schema tag
  `0x446E735172797201` = "DnsQry"+ver01), prints `pdxdig: dry-run
  ok\n` (19 wire bytes) on fd 2, and exits 0. The record is the
  emission channel the plan §10.2 specifies -- "prints the record
  it would emit" in the semantic-pipe world means "emits on the
  semantic pipe with result_code = DRY_RUN so a consumer sees
  the intended query without a network round-trip."

- **UNSUPPORTED_QTYPE handling (pdxdig#6, M2-003).** `--type=<X>`
  where `X` is anything other than exactly `A` (Main::_start byte-
  checks arg[7]=='A' + arg[8]==NUL) sets rbx bit 1, marshals the
  record with `pdxdig_dqr_tail_unsupp` (result_code = 4,
  UNSUPPORTED_QTYPE), emits via `sys_semantic_send`, prints
  `pdxdig: unsupported qtype\n` (26 wire bytes) on fd 2, and
  exits with status 4 -- a distinct exit code from a real DNS
  failure (exit 1 for parser mismatches, exit 3 for the UDP stub).
  Precedence: --type=<non-A> beats --dry-run (a rejected --type=
  is a client-side error regardless of --dry-run).

### Recorded (honest witnesses -- no code change)
- **Scaffold + KIND_USER caps.decl (pdxdig#1, M1-001).** The
  v1.1.0 landing already carried `src/main.pdx`, `src/dns_walker.
  pdx`, `src/tool_ident.pdx`, `manifest.pdxproj`, `link.ld`,
  `deps.list`, and `caps.decl` (with `!KIND_USER 0x001` mandatory
  + `KIND_UDP_SOCKET 0x00B` and `KIND_IPC_ENDPOINT 0x001`
  optional). Wave W formally records this as closed rather than
  leaving the M1-001 issue drifting.

- **Semantic-pipe schema bind + emit (pdxdig#8, M3-002).** The
  DnsQueryRecord@0.1 schema fingerprint constant
  (`0x446E735172797201`), the 48-byte record layout, and the
  `sys_semantic_send` emit wire already landed at Wave L v1.1.0
  under pdxdig#17 (v1.1-B). The record's txn_id slot lives at
  byte offset [+43..+44] within the tail u64 (visible in the
  layout comment in `src/main.pdx`); at v1.2 the slot stays 0 --
  a real per-query u16 randomization gates on the M2-001 UDP
  substrate (§2.3.1). Wave W records the schema-bind + emit
  contract formally as closed.

## [1.1.0] -- 2026-09-13 -- Wave L drain

### Added
- **Source scaffold + first-real-body tool.** Retires the M1-001
  README-only seed. The satellite now has `src/main.pdx` (module
  `Main`) with the `_start` argv parser + `pdxdig <name>` positional
  gate + the v1.1-B semantic-pipe emission wire; `src/dns_walker.pdx`
  (module `DnsWalker`) with the real response-header RCODE parser
  and the CNAME-follow loop (loop-guard at 4 hops); and
  `src/tool_ident.pdx` (module `ToolIdent`) publishing
  `PDX_TOOL_NAME = "pdxdig\0"` and `PDX_TOOL_VERSION = "1.1.0\0"`
  per the libpdx-argv 1.1.3 ENH-032 UND-extern contract.

- **v1.1-B semantic-pipe emission wire (pdxdig#17).** After each
  argv-accepted invocation the tool marshals a
  `DnsQueryRecord@0.1` (48 bytes, `@align(8)`; leading `PDXKDIG_`
  ASCII magic + version/flags header + audit_id + qname_hash +
  qtype + answer_ip + ttl + cname_hops + txn_id + result_code +
  ts_ns per §10.2 layout) into `pdxdig_record_buf` and emits it
  via `sys_semantic_send` (SC+ ID 115, schema tag
  `0x446E735172797201` -- "DnsQry" + version 0x01). The real UDP
  query path is BLOCKED on §12.1 (SOCK_DGRAM extension); at
  v1.1.0 the marshal fabricates a `result_code = DRY_RUN` record
  and the tool exits 3 with `pdxdig: udp stub\n` on fd 2. The
  emit wire itself is real; a follow-up M2-001 landing flips
  the classifier to record `OK` / `NXDOMAIN` / `TIMEOUT` /
  `MALFORMED_RESPONSE` / `UNSUPPORTED_QTYPE` per the query
  outcome without touching the marshal or send-syscall layout.

- **NXDOMAIN smoke witness (pdxdig#10, M4-002).** Adds
  `tests/m4_002_nxdomain.pdx` (module `M4002Nxdomain`) as a
  self-contained ELF that constructs an in-.rodata DNS response
  header with `RCODE = 3` (NXDOMAIN), invokes the shared
  `DnsWalker::dns_walker_rcode` parser, and asserts the low nibble
  of `flags[3]` equals 3. Fingerprint band on fd 2: `pdxdig
  nxdomain ok\n` (20 wire bytes) on success, `pdxdig nxdomain
  FAIL\n` (22 wire bytes) on parser mismatch. Exits 0 on success,
  1 on parser mismatch. Gates the RCODE-parse contract without
  needing a live resolver.

- **Timeout smoke witness (pdxdig#11, M4-003).** Adds
  `tests/m4_003_timeout.pdx` (module `M4003Timeout`). Because the
  real UDP query path is blocked on §12.1, the witness constructs
  the timeout classifier's expected exit-code + fingerprint band
  synthetically: it invokes the shared classifier with a
  `sentinel_response_len = 0` (no bytes received in the
  timeout-ms window), asserts the classifier reports
  `result_code = TIMEOUT`, emits `pdxdig timeout ok\n` (19 wire
  bytes) on fd 2, and exits 4 (the tool-reserved TIMEOUT status).
  A follow-up M4-003-b landing wires the same classifier over a
  real 100ms sleep once sys_clock_now + a blocking sys_recv with
  a real deadline exist.

- **Malformed-response fuzz-derived fixture matrix (pdxdig#12,
  M4-004).** Adds `tests/m4_004_malformed_matrix.pdx` (module
  `M4004MalformedMatrix`) driving the shared response walker
  through five fixtures: `mm_trunc_hdr` (7-byte truncated
  header), `mm_bad_arcount` (declared 4 answers, zero bytes of
  answer section), `mm_ptr_oob` (compression pointer 0xC0FF ->
  offset past response end), `mm_dangling_label` (length byte
  0x40 with no following bytes), `mm_junk_trailer` (well-formed
  12-byte header + 0xFF junk). Each fixture calls
  `DnsWalker::dns_walker_validate` and asserts a negative
  `-EINVAL` return without a wild read. Emits `pdxdig malformed
  ok idx=<N>\n` per accepted refusal; final `pdxdig malformed all
  ok\n` + exit 0. Any wild-read fixture (walker returns >= 0)
  emits `pdxdig malformed FAIL idx=<N>\n` and exits 1.

- **CNAME follow correctness with 4-hop loop-guard (pdxdig#13,
  M4-005).** Implements real CNAME resolution in
  `DnsWalker::dns_walker_follow_cname`: one-hop A resolution
  succeeds, chains of length 2..4 succeed, chains of length 5+
  (or CNAME->CNAME->...->self loops) trip the guard and return
  `-ELOOP`. Adds `tests/m4_005_cname_smoke.pdx` (module
  `M4005CnameSmoke`) with two fixtures: `cs_one_hop` (CNAME
  target -> A record; walker returns hops=1, answer_ip = fixture
  value) and `cs_loop_5` (5-hop chain; walker returns -ELOOP).
  Emits `pdxdig cname ok hops=1\n` + `pdxdig cname loop
  guard=4\n` on success; either mismatch emits `pdxdig cname
  FAIL\n` and exits 1.

### Infrastructure
- `.gitignore` mirroring pdxsock's (build-out/, *.o, *.bin, *.elf,
  *.a, *.so, editor + OS noise, `.scratch/`).
- `manifest.pdxproj` at `version = 1.1.0` (Wave L drain closer);
  `tools/build.sh` mirroring pdxsock's paideia-as->ld pipeline
  emitting `build-out/pdxdig.elf` + `pdxdig.bin`; `link.ld` with
  `_start` entry + `.text` at `0x00400000` + `.data`/.bss at
  `0x00600000`; `caps.decl` declaring `!KIND_USER` mandatory and
  `KIND_UDP_SOCKET` + `KIND_IPC_ENDPOINT` optional; `deps.list`
  documenting the deferred libpdx-net / libpdx-argv wire-ins.

### Design notes
- **Semantic-pipe magic + schema tag.** The record's leading
  8-byte magic is `PDXKDIG_` (ASCII, LE u64
  `0x5F474944584450`); the schema-tag u64 written to the
  `sys_semantic_send` argument is `0x446E735172797201` ("DnsQry"
  + version 0x01). Both are grep-friendly ad-hoc constants
  preserved verbatim once paideia-os#2000 (schema registry)
  lands. This is intentional shape-parity with the
  `SockSessionRecord@0.1` (`PDXKSOCK`, `0x536F636B53657301`)
  precedent -- consumers select record family by the leading
  magic and version fields alone.

- **Fixture-harness pattern.** Every test compiles today; each is
  a single-role or dual-role self-contained ELF. Runtime round-
  trip witnessing is deferred to the paideia-os smoke-runner
  extension (matches the pdxsock M4-001..M4-003 pattern -- see
  `pdxsock/tests/tcp_echo_smoke.pdx` header for the discipline
  rationale). The `tools/build.sh` under the `tests/*.pdx` glob
  emits `build-out/tests-m4_00N_<name>.o` only; the paideia-os
  monorepo picks these up via the /bin seeding pipeline (issues
  #1976/#1977).
