# Changelog

All notable changes to `pdxdig` are recorded here. Format: keep-a-
changelog-style, semver-ordered, newest first.

<!--
Version discipline:
  v1.0.x -- reserved for the original M5 dual-signed 1.0-shape plan.
             No v1.0.0 tag was ever cut.
  v1.1.0 -- Wave L drain: source scaffold + first-real-body tool
             + tests/*.pdx witnesses (pdxdig#10, #11, #12, #13, #17).
-->

## [Unreleased]

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
