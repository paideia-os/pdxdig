# pdxdig

DNS query CLI (`dig`-equivalent). `A`-record only at v1 (`libpdx-net`'s resolver parses no other RR type); `--type=` beyond `A` returns a clean `UNSUPPORTED_QTYPE`. `--server=` overrides the boot-seeded resolver, `--timeout-ms` and `--dry-run` supported. No elevate needed — UDP DNS queries take no more privilege than the TCP sockets other tools already use.

## Spec

Full design lives in the paideia-os monorepo at
[`design/networking/r100-user-tools-plan.md`](https://github.com/paideia-os/paideia-os/blob/main/design/networking/r100-user-tools-plan.md)
(softarch's R100 user-tools plan). Section references in issues point into that document.

This repository is one of seven satellite repos that together deliver the
R100 wave: `libpdx-net`, `libpdx-url`, `pdxcurl`, `pdxping`, `pdxdig`,
`pdxsock`, `pdxtrust`.

## License

MIT.