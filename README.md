# reticulum-go

> **This is a fork of [`thatSFguy/reticulum-go`](https://github.com/thatSFguy/reticulum-go).**
>
> **Its purpose is to make `fwdsvc` usable over HF.** `reticulum-go` sizes its link and
> delivery timeouts for LoRa, and a half-duplex HF link needs far longer. That is what
> this build is for; the `[timing]` and `[outbound]` sections below are the point.
>
> It also carries a second, unrelated fix: upstream rejects the propagation-node announce
> that current Python LXMF publishes, which breaks store-and-forward uploads against every
> modern propagation node, including a stock `lxmd`. That was found while working on the
> timeouts and is included because it blocks the same program.
>
> | | |
> |---|---|
> | Fork point | `v0.8.1` (`2dcc98c`), upstream's newest tag |
> | Files touched | `lxmf/propagation.go`, `lxmf/propagation_test.go` |
> | Program it was built for | [`fwdsvc`](https://github.com/thatSFguy/reticulum-group-chat) |
> | Everything else | unmodified upstream |
>
> Nothing has been reported upstream — that is deliberate.

## Purpose: link and retry timeouts for HF

`reticulum-go` sizes its delivery timeouts for LoRa, and they are too short for a half-duplex
HF link. On HF a single frame occupies the channel for over ten seconds, so a round trip is
measured in minutes, not seconds.

Defaults in the library:

| Constant | Value | Covers |
|---|---|---|
| `rns.DefaultLinkSendTimeout` | **30s** | handshake + DATA broadcast + proof wait, one direct send |
| `lxmf.DefaultDeliveryProofTimeout` | **15s** | wait for the recipient's delivery proof |
| `lxmf.DefaultPropagationSendTimeout` | **45s** | one propagation-node upload |
| `maxDeliveryAttempts` | **5** | direct-delivery attempts before falling back |
| `deliveryRetryWait` | **10s** | pause between attempts |

Measured round trips on this network: **35.7s, 40.5s, 120.7s**. A 30s ceiling cannot complete a
send whose round trip is 40s — the failure is arithmetic, not bad luck. And a 10s retry on a
channel where a single frame lasts over ten seconds starts before the previous attempt's airtime
has cleared.

Two config sections override them. **Both are optional, and both leave the library default when
absent or zero**, so a fast member on TCP pays nothing.

```toml
[timing]
# Cap for one direct send: handshake + DATA broadcast + proof wait.
link_send_timeout       = "240s"
# Cap for waiting on a recipient's delivery proof.
delivery_proof_timeout  = "90s"
# Cap for a single propagation-node upload. Default 45s is too short when
# the node is reached over the same slow link; leave unset when it is local.
# propagation_send_timeout = "240s"

[outbound]
# Attempts before a message is dropped or handed to a propagation node.
max_attempts       = 2
# Pause between attempts. Must exceed the slowest observed success.
retry_wait         = "180s"
# Optional ceiling for exponential backoff between attempts.
# retry_max_backoff = "600s"
```

### Why the attempt count matters as much as the timeout

`outboundWorkers = 8`, and each in-flight send holds one worker for the whole timeout. Five
attempts at 240s against three unreachable members is an hour of saturated workers; two attempts
is far less. Raising `link_send_timeout` without lowering `max_attempts` trades one failure mode
for another.

### These only help the initiator

Fan-out to each member is a link **this host initiates**, so this host's budget decides the
outcome. **No setting on the remote end can compensate.** Set these where the slow members are
dialled from — the hub — not on the members.

### Where the values come from

`retry_wait` must exceed the slowest observed successful establishment (120.7s here), otherwise a
retry begins while the previous attempt is still on the air. `link_send_timeout` must exceed the
slowest round trip including proof. Both are measurements, not guesses, and both are specific to
the link they were taken on — **re-measure before copying them to a different band or mode.**

## Secondary fix: propagation-node announces

A Reticulum propagation node advertises itself in an announce whose `app_data` is a msgpack
list. LXMF 1.3.0 appends an eighth element holding fractional stamp costs, and validates it as
`len(data) > 7` with a 3-element numeric list — marked `TODO: Transitional mechanism, remove
after 1.6.2` in Python's `pn_announce_data_is_valid`.

`reticulum-go v0.8.1` required exactly seven:

```go
if len(elems) != 7 {
    return nil, fmt.Errorf("%w: %d elements, want 7", ErrPropagationAppData, len(elems))
}
```

So a Go client refused every current Python propagation node, **including a stock `lxmd` on the
same machine**, and failed at the announce before any transfer began:

```
propagation: upload failed: invalid lxmf.propagation announce app_data: 8 elements, want 7
```

Direct delivery was unaffected, so the symptom read as "store-and-forward is down" rather than
"the network is broken".

This fork accepts `len(elems) >= 7` and, when element `[7]` is present, requires a 3-element list
of numerics. A malformed `[7]` still fails with `ErrPropagationAppData`. Elements `[3]`/`[4]`
limits and `[5]` integer costs are untouched.

## How to use it

In each Go module that imports `reticulum-go`:

```
go mod edit -replace github.com/thatSFguy/reticulum-go=github.com/smeshT/fwdsvc_hf@<commit>
go mod tidy
go build ./...
```

**Pin a commit hash, not a branch name** — a branch moves, a dependency should not change under a
build. `<commit>` is the head of `fix/propagation-announce-8-elements`.

If the remote is not yet configured in that clone:

```
git remote set-url origin git@github.com:smeshT/fwdsvc_hf.git
```

Confirm the replace resolved:

```
go list -m -f '{{.Path}} => {{if .Replace}}{{.Replace.Path}} {{.Replace.Version}}{{else}}NONE{{end}}' github.com/thatSFguy/reticulum-go
```

Expected output:

```
github.com/thatSFguy/reticulum-go => github.com/smeshT/fwdsvc_hf v0.0.0-2026...-<commit>
```

Import statements never change. `go.mod` keeps the upstream module path; `replace` redirects it.

## Installing

This repository is the library fork. Building a program against it needs the `replace` from
[How to use it](#how-to-use-it); nothing here installs a binary by itself.

For the program this build was made for — `fwdsvc`, from
[`reticulum-group-chat`](https://github.com/thatSFguy/reticulum-group-chat):

```bash
# 1. Point the program at this fork (once, from the program's module root)
cd ~/src/reticulum-group-chat
go mod edit -replace github.com/thatSFguy/reticulum-go=github.com/smeshT/fwdsvc_hf@<commit>
go mod tidy

# 2. Build
go build -o /tmp/fwdsvc ./cmd/fwdsvc

# 3. Install, keeping the previous binary
install -m 755 /home/jack/.local/bin/fwdsvc /home/jack/.local/bin/fwdsvc.bak-$(date +%Y%m%d-%H%M%S)
install -m 755 /tmp/fwdsvc /home/jack/.local/bin/fwdsvc

# 4. Restart the instances (templated units, one per room)
sudo systemctl restart fwdsvc@thenet fwdsvc@gcc fwdsvc@mtnway
```

Verify the running process is the new binary, not the old one still mapped from a deleted inode:

```bash
ls -l /proc/$(pgrep -f '/home/jack/.local/bin/fwdsvc' | head -1)/exe
```

A path ending in `(deleted)` means the file on disk was replaced but the process has not
restarted — the fix is **not** active yet.

### Requirements

Go 1.26.1+ to build. No cgo, no third-party RNS library.

## Where the fix goes — and where it does not

A `replace` is **per Go module**, not per program and not per machine.

| Component | Language | Needs the replace? |
|---|---|---|
| `reticulum-group-chat` (builds `fwdsvc`) | Go | **yes** |
| `reticulum-relay-chat` | Go | yes, if run and it imports `reticulum-go` |
| `semaphore` | Go | yes, if run and it imports `reticulum-go` |
| `rnsd`, `lxmd`, `nomadnet`, MeshChatX | Python | **no** |

**The Python side is unaffected in both directions.** It already accepts 8 elements — which is
exactly why a Python client could upload to the node while `fwdsvc` could not. There is no
"instance" to patch; the unit is the module.

**Commit the `replace`.** An uncommitted `go.mod` change means a fresh clone silently builds
against stock upstream and the bug returns with no error. `git checkout go.mod` or `git stash`
removes it the same silent way.

## Verifying the fix

A passing test proves little on its own, since a test can be written to match its own fix. The
useful check is the reverse — revert the source, keep the tests:

```
git show v0.8.1:lxmf/propagation.go > lxmf/propagation.go
go test ./lxmf/ -run FractionalStampCosts -v
```

Expected, reproducing the field error verbatim against unmodified upstream:

```
8-element announce rejected: invalid lxmf.propagation announce app_data: 8 elements, want 7
```

Restore with `git checkout lxmf/propagation.go`.

## Maintenance

If upstream accepts 8 elements, drop the `replace` and delete this fork. The diff is 32 lines of
source plus tests, so rebasing against a rewritten parser would be trivial to redo.

Upstream has not fixed this as of `v0.8.1` (`2dcc98c`), the newest tag — an annotated tag whose
commit is also upstream `main`, so there is no newer code to rebase onto. Whether to accept or
reject the eighth element belongs to its author; this fork accepts it because that is what the
network currently sends.

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
