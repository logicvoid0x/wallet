# wallet

A terminal Bitcoin wallet — planned as a **watch-only PSBT coordinator** that never holds a
private key.

> ### ⚠️ Status: design phase. There is no code in this repository.
>
> This repo currently contains a README and nothing else. Nothing here is usable, and
> nothing here has been tested. **Do not point this at real funds** — there is nothing to
> point.

---

## What this is going to be

A terminal-UI Bitcoin wallet in the shape of [Sparrow](https://sparrowwallet.com/) or
[Specter](https://specter.solutions/), but driven from a TUI rather than a desktop GUI.

The defining constraint: **hardware signers only.** The wallet holds no private keys, no seed
phrases, and no signing code. It builds PSBTs, hands them to a hardware device, and takes
back the signed result. That design removes the risk class that makes self-custody software
frightening to ship — there is no hot key to steal, because there is no hot key.

Planned capabilities:

- Single-sig and n-of-m multisig, driven by [output descriptors](https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki)
- PSBT construction and hardware-signer coordination
- Coin control, fee estimation, RBF/CPFP
- Chain data via Esplora/Electrum, **with user-configurable endpoints** — pointing at your
  own instance is a requirement, not a nice-to-have, because a third-party server otherwise
  sees your entire address set

## First component: `wallet-descriptors`

Work has started on the piece everything else sits on: a reusable Rust crate that turns
output descriptors and xpubs into derived addresses, for single-sig and multisig.

It is the seam beneath the whole wallet — storage persists descriptors, the UI renders
addresses derived from them, coin control maps UTXOs back to derivation paths, and change
detection is the same operation on the internal chain. So it gets specced, built and verified
on its own before anything is built on top of it.

Two facts shape its design:

- **A bare xpub does not encode a script type.** `xpub…` alone cannot tell you whether to
  emit `1…`, `3…`, `bc1q…` or `bc1p…`. SLIP-132 prefixes (`ypub`/`zpub`) encode it but are
  non-standard and discouraged.
- **Multisig needs more than xpubs carry** — threshold, key ordering (`sortedmulti` vs
  `multi`, per [BIP-67](https://github.com/bitcoin/bips/blob/master/bip-0067.mediawiki)),
  script wrapper, and a derivation path per key.

Both are exactly what output descriptors exist to express. So descriptors are the canonical
input, and any xpub convenience mode is a thin wrapper that assembles a descriptor from
explicit flags and echoes it back for you to verify and save.

## Design decisions settled so far

| Decision | Choice |
|---|---|
| Custody | Self-custody; hardware signers only, no keys in software |
| Form factor | Terminal UI (Rust + [ratatui](https://ratatui.rs/)) |
| Chain data | Esplora/Electrum, user-configurable endpoint |
| Descriptor layer | Bare [`rust-miniscript`](https://github.com/rust-bitcoin/rust-miniscript) + `bitcoin` — deliberately *not* BDK |
| Scope | Single-sig and multisig; full-featured (multisig, PSBT, coin control, RBF, labels) |

### Why not BDK?

[BDK](https://bitcoindevkit.org/) is excellent and this project may still use it in the
wallet layer above. But the descriptor crate sits *below* it, for three measured reasons:

1. `bdk_wallet` 3.1.0 pins `miniscript ^12.3.5`. Depending on both puts two incompatible
   `Descriptor` types in the dependency graph, and 12.x lacks the rewritten Taproot API.
2. Dependency surface is **12 packages** on bare `miniscript` versus **34** via `bdk_wallet`,
   including build-time proc macros. For software guarding real money, the dependency graph
   *is* the attack surface.
3. BDK ships signing code, BIP-39, and `DescriptorSecretKey` handling — all of which a
   watch-only crate has promised never to use. On bare miniscript,
   `Secp256k1::verification_only()` **cannot sign at all**, which turns that promise into a
   compiler-enforced guarantee.

## Known ecosystem constraints

Research against `miniscript` 13.1.0 and `bdk_wallet` 3.1.0 turned up three things that
directly limit what can ship:

- **`sortedmulti_a` is absent from every released `rust-miniscript`.** It is defined by
  [BIP-387](https://github.com/bitcoin/bips/blob/master/bip-0387.mediawiki) and deployed in
  Bitcoin Core since 24.0, so taproot-multisig descriptors exported by Core, Sparrow, or
  hardware signers **will fail to parse**. It is merged to `master` and queued for a future
  release.
- **musig2 does not exist** in either library. BIP-390 is still Draft. Not immature — absent.
- **Network validation is the caller's job.** Descriptors are network-free, and
  `.address(Network::Bitcoin)` on a `tpub` descriptor will happily emit a mainnet address
  from testnet keys. `xkey_network()` only *reports*; it does not enforce.

> These findings come from reading published crate sources, not from running code. They are
> well-sourced but **unverified by execution**, and re-verifying them is the first task once
> a build toolchain is in place.

## Roadmap

- [x] Scope and design the descriptor derivation module
- [x] Research the `rust-miniscript` / BDK capability landscape
- [ ] Settle accepted input forms and supported script policy
- [ ] Settle the module's public API and crate boundary
- [ ] Settle correctness strategy and test vectors
- [ ] Settle dependency version and upgrade policy
- [ ] Write the spec, then implement `wallet-descriptors`
- [ ] Design the wallet coordinator (hardware integration, TUI, PSBT flows) — a separate
      effort that begins once the descriptor crate exists

Planning artifacts live outside version control and are not part of this repository.

## Security

This project intends to handle real funds eventually, and it isn't there yet. When there is
code, it will be watch-only and will never hold a private key.

Please **do not** use anything from this repository with mainnet funds until a release
explicitly says it is ready. A responsible-disclosure policy will be published alongside the
first release.

## License

Not yet chosen. Until a license file is added, no license is granted.
