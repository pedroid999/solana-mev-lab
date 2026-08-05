# solana-mev-lab

An [Anchor](https://www.anchor-lang.com/) workspace on Solana, intended as a lab for
experimenting with on-chain programs. The current program is a small **Counter**: a
PDA-backed account with an authority check and a maximum value. It's the scaffold the
lab builds on — there is no MEV-specific logic in the program yet.

## What the program does

The `solana_mev_lab` program exposes two instructions:

| Instruction  | What it does |
| ------------ | ------------ |
| `initialize` | Creates the `Counter` PDA (seed `"counter"`), sets `count = 0`, records the payer as `authority`, and transfers `HELLO_WORLD_LAMPORTS` (1 lamport) from the payer into the counter account. |
| `increment`  | Increments `count` by 1. Requires the signer to be the recorded `authority` and fails once `count` reaches `MAX_COUNT` (10). |

State (`Counter`):

```rust
pub struct Counter {
    pub count: u64,
    pub authority: Pubkey,
}
```

Errors:

- `Unauthorized` — signer is not the counter's authority.
- `CounterOverflow` — counter already at `MAX_COUNT`.

Program ID: `4qZFzzWnb8VHLRV7UzWQRQEKGycMchHpPxEGWssZR8Zu` (localnet)

## Toolchain versions

These are the exact versions this repo is developed and tested against:

```text
rustc                  1.89.0 (29483883e 2025-08-04)
cargo                  1.89.0 (c24e10642 2025-06-23)
solana-cli             4.1.1 (Agave, src:6a8c724a; feat:c763ae0a)
anchor-cli             1.1.2
solana-test-validator  4.1.1 (Agave, src:6a8c724a; feat:c763ae0a)
```

Rust is pinned via `rust-toolchain.toml` (channel `1.89.0`, with `rustfmt` and `clippy`).
The Solana CLI is the Agave client. Anchor `1.1.2` is used both as the CLI and as the
`anchor-lang` dependency.

## Project layout

```text
.
├── Anchor.toml                       # Anchor config (localnet, program id, test script)
├── Cargo.toml                        # Workspace root (edition 2021, release profile)
├── rust-toolchain.toml               # Pinned Rust toolchain (1.89.0)
├── app/                              # Client app placeholder (empty)
└── programs/
    └── solana-mev-lab/
        ├── Cargo.toml
        ├── src/
        │   ├── lib.rs                # declare_id! + #[program] entrypoints
        │   ├── constants.rs          # COUNTER_SEED, HELLO_WORLD_LAMPORTS, MAX_COUNT
        │   ├── error.rs              # ErrorCode enum
        │   ├── state.rs              # Counter account
        │   ├── instructions.rs       # instruction module re-exports
        │   └── instructions/
        │       ├── initialize.rs
        │       └── increment.rs
        └── tests/
            └── test_initialize.rs    # LiteSVM integration test
```

## Prerequisites

- Rust `1.89.0` (installed automatically from `rust-toolchain.toml` if you use rustup)
- Solana CLI (Agave) `4.1.1`
- Anchor CLI `1.1.2`
- A local wallet keypair at `~/.config/solana/id.json`

## Build

```bash
anchor build
```

This compiles the program to `target/deploy/solana_mev_lab.so`. The integration test
loads that `.so` directly, so **you must build before running the tests**.

## Test

Tests use [LiteSVM](https://github.com/LiteSVM/litesvm) — an in-process Solana VM — so
no validator needs to be running. The test loads the compiled program, airdrops to a
payer, runs `initialize`, asserts the initial state, then runs `increment` and asserts
`count == 1`.

```bash
anchor build        # produces the .so the test includes via include_bytes!
cargo test
```

`Anchor.toml` maps `anchor test` to `cargo test` and sets `skip_local_validator = true`,
since LiteSVM replaces the validator.

## Notes

- `overflow-checks = true` is enabled in the release profile, so arithmetic overflows
  abort rather than wrap.
- `initialize` moves 1 lamport into the counter PDA on top of the rent-exempt minimum —
  a placeholder for future logic, not a functional requirement of the counter.
- The `app/` directory is an empty placeholder for a future client.
