# Using this fork

A fork of [`thatSFguy/reticulum-go`](https://github.com/thatSFguy/reticulum-go) carrying a
single fix. Everything else is upstream, unmodified.

## Why this exists

Propagation nodes announce themselves with an `app_data` payload. Upstream LXMF appends an
eighth element to that payload, carrying fractional stamp costs:

```python
# LXMF.py, pn_announce_data_is_valid
if len(data) < 7: raise ValueError(...)

# TODO: Transitional mechanism, remove after 1.6.2
if len(data) > 7:
    if type(data[7]) != list: raise ValueError(...)
    # fractional: target stamp cost, stamp cost flexibility, peering cost
```

Upstream `pn_announce_data_is_valid` therefore requires **at least** 7 elements and validates
the optional eighth. `reticulum-go v0.8.1` required **exactly** 7:

```go
if len(elems) != 7 {
    return nil, fmt.Errorf("%w: %d elements, want 7", ErrPropagationAppData, len(elems))
}
```

The result is that a Go client cannot upload to any current Python propagation node — including
a stock `lxmd`, on the same machine. The failure appears as

```
propagation: upload failed: invalid lxmf.propagation announce app_data: 8 elements, want 7
```

and it fails at the announce, before any transfer begins. Direct (non-propagated) delivery is
unaffected, which is why the symptom looks like "store-and-forward is down" rather than
"everything is down".

## What changed

One branch, `fix/propagation-announce-8-elements`, one commit on top of `v0.8.1`:

- `lxmf/propagation.go` — accept `len(elems) >= 7`; when element `[7]` is present, require a
  3-element list of numerics; a malformed `[7]` still fails with `ErrPropagationAppData`.
  Elements `[3]`/`[4]` limits and `[5]` integer costs are untouched.
- `lxmf/propagation_test.go` — a test built from the announce a stock `lxmd` actually produces,
  plus tests for the malformed shapes that must still be rejected.

## Use it

In your module:

```
go mod edit -replace github.com/thatSFguy/reticulum-go=github.com/smeshT/fwdsvc_hf@<commit>
go mod tidy
```

A commit hash is used rather than a branch name so the dependency cannot change underneath a
build. Replace `<commit>` with the head of `fix/propagation-announce-8-elements`.

To confirm the replace took effect:

```
go list -m -f '{{.Path}} => {{.Replace.Path}} {{.Replace.Version}}' github.com/thatSFguy/reticulum-go
```

`go.mod` keeps the upstream import path. No import statements change — a `replace` redirects the
module without renaming it.

## Verify

```
go test ./lxmf/ -run PropagationNodeAppData -v
```

Passing tests alone do not prove much, since a test can be written to match its own fix. The
useful check is the reverse: with `lxmf/propagation.go` reverted to `v0.8.1` and the tests kept,
`TestParsePropagationNodeAppDataToleratesFractionalStampCosts` fails with

```
8-element announce rejected: invalid lxmf.propagation announce app_data: 8 elements, want 7
```

which is the field error reproduced verbatim on unmodified upstream.

## Status

- **Not reported upstream.** This fork is a local workaround, not a claim about whether
  `reticulum-go` or Python LXMF is "correct". Upstream's own transitional comment suggests the
  eighth element is intended to disappear after LXMF 1.6.2, so a decision there belongs to its
  author.
- **Upstream has not fixed this as of `v0.8.1` (`2dcc98c`)**, the newest tag. It is an annotated
  tag whose commit is also upstream `main` (`git rev-parse v0.8.1^{commit}`), so there is no
  newer upstream code to rebase onto.
- **`main` in this fork is untouched upstream.** The fix lives only on its own branch.

## Maintenance

If upstream fixes this, drop the `replace` and delete this fork; nothing else depends on it.
If upstream moves on first, rebase the branch — the diff is 32 lines of source plus tests, so
conflicts should be trivial unless the parser has been rewritten.
