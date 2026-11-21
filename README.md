# reticulum-go

> **This is a fork of [`thatSFguy/reticulum-go`](https://github.com/thatSFguy/reticulum-go).**
>
> It exists for **one reason**: upstream rejects the propagation-node announce that
> current Python LXMF publishes, which makes store-and-forward uploads fail against
> every modern propagation node, including a stock `lxmd`.
>
> | | |
> |---|---|
> | Fork point | `v0.8.1` (`2dcc98c`), identical to upstream `main` |
> | Fix branch | `fix/propagation-announce-8-elements` |
> | Files touched | `lxmf/propagation.go`, `lxmf/propagation_test.go` |
> | Everything else | unmodified upstream |
>
> Use the fix branch; `main` here is the untouched upstream commit. Details and the
> one-line `go.mod` change needed to depend on this fork are in
> [FORK.md](FORK.md). Nothing has been reported upstream — that is deliberate.

Pure-Go implementation of the [Reticulum Network Stack](https://reticulum.network/) (RNS) and [LXMF](https://github.com/markqvist/LXMF) — no third-party RNS library, no cgo. Interoperates with the canonical Python `RNS`/`LXMF` and the wider client ecosystem (Sideband, NomadNet, MeshChat).

```go
import (
    "github.com/thatSFguy/reticulum-go/rns"
    "github.com/thatSFguy/reticulum-go/lxmf"
)
```

## Packages

| Package | Contents |
|---|---|
| `rns` | Identity & destinations, packet/announce wire formats, links (handshake, tokens, state), transport & path-finding, proofs, resources (sender/receiver/ADV), HDLC + TCP interfaces. |
| `lxmf` | LXMF messages, opportunistic & link delivery, delivery proofs, propagation (store-and-forward), stamps/tickets. Depends only on `rns`. |

Wire formats are tracked against [`reticulum-specifications`](https://github.com/thatSFguy/reticulum-specifications) (byte-level reference, with runtime verifiers). The test suites include upstream interop fixtures under each package's `testdata/`.

## Provenance

Extracted from the RNS/LXMF core shared by [`reticulum-group-chat`](https://github.com/thatSFguy/reticulum-group-chat) and [`reticulum-relay-chat`](https://github.com/thatSFguy/reticulum-relay-chat), reconciled into one canonical module (group-chat's newer stack as the base, with relay-chat's unique hardening forward-ported). Those repos, and [`semaphore`](https://github.com/thatSFguy/semaphore), now depend on this module rather than carrying their own copy.

## Requirements

Go 1.26.1+. Dependencies: `golang.org/x/crypto`, `github.com/vmihailenco/msgpack/v5`.

## License

[MIT](LICENSE) © 2026 thatSFguy
