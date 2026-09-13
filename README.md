# pdxdig

DNS query CLI (`dig`-equivalent). `A`-record only at v1 (`libpdx-net`'s resolver parses no other RR type); `--type=` beyond `A` returns a clean `UNSUPPORTED_QTYPE`. `--server=IP.IP.IP.IP` parses and stores an IPv4 override (a malformed value is a usage refusal); `--timeout-ms` and `--dry-run` supported. No elevate needed — UDP DNS queries take no more privilege than the TCP sockets other tools already use.

The real UDP query round-trip is BLOCKED on the kernel-side SOCK_DGRAM extension (§12.1 of the design doc below, tracked as R100-PREP-002): every invocation today marshals and emits a `DnsQueryRecord@0.1` on the semantic pipe with a `DRY_RUN` / `UNSUPPORTED_QTYPE` result code, and the no-flag path exits 3 (`pdxdig: udp stub`). `src/dig_query.pdx` documents the real `net_resolve` extern this will call once the dep lands; `src/audit_wire.pdx` documents the real `libpdx-audit` fingerprint call the same is true of.

## Spec

Full design lives in the paideia-os monorepo at
[`design/networking/r100-user-tools-plan.md`](https://github.com/paideia-os/paideia-os/blob/main/design/networking/r100-user-tools-plan.md)
(softarch's R100 user-tools plan). Section references in issues point into that document.

This repository is one of seven satellite repos that together deliver the
R100 wave: `libpdx-net`, `libpdx-url`, `pdxcurl`, `pdxping`, `pdxdig`,
`pdxsock`, `pdxtrust`.

## License

MIT.