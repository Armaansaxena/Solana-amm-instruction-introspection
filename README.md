# Solana AMM with Instruction Introspection (Anchor)

A constant-product AMM (`x * y = k`) on Solana, written in Rust with Anchor. Withdrawals are split into two instructions, `burn_lp` and `payout`, and `payout` uses instruction introspection to verify that a matching burn happened immediately before it in the same transaction. Built during the Turbin3 Builders Cohort.

- **Stack:** Rust, Anchor, TypeScript (tests)
- **Status:** Educational project, not audited. Do not use with real funds.

## Features

- Pool creation, liquidity deposits and token swaps
- Pool lock and unlock guards (deposits fail while the pool is locked)
- Split withdrawal: `burn_lp` burns the user's LP tokens, `payout` returns the proportional share of the reserves
- Instruction introspection through the `Instructions` sysvar, so a payout cannot happen without a preceding burn
- TypeScript test suite covering the main flows and the failure cases

## How withdrawal works

The withdrawal is two program instructions in one transaction:

1. **`burn_lp`** burns the user's LP tokens and records the amount in its instruction data.
2. **`payout`** verifies the burn and transfers the proportional share of the pool's reserves to the user.

In `payout`, the program reads the `Instructions` sysvar and:

- loads the index of the current instruction,
- fetches the instruction immediately before it (`index - 1`),
- checks that its `program_id` is this program and its discriminator is `burn_lp`,
- reads the burned `lp_amount` from that instruction's data,
- checks that the `user` and `pool` accounts match the current context, so a burn for a different user or pool cannot be reused.

Because both instructions are in the same transaction, they succeed or fail together. A `payout` without a matching `burn_lp` right before it is rejected, and the payout amount is tied to the amount burned.

## Tests

The suite is in `tests/amm_tests.ts`. All 8 tests pass:

1. Initializes the pool
2. Deposits liquidity (first deposit)
3. Swaps token0 for token1
4. Withdraws liquidity (split instructions with introspection)
5. Fails to payout without a burn instruction
6. Fails to payout if the burn instruction is in the wrong order
7. Locks the pool and fails to deposit
8. Unlocks the pool and allows deposit


![Tests Passing](./test-passing-2.png)


## Project structure

```
programs/amm/src/
├── lib.rs                  # Program entrypoint and routing
└── instructions/
    ├── burn_lp.rs          # Burns LP tokens, records the amount
    └── payout.rs           # Verifies the burn via introspection, pays out
tests/
└── amm_tests.ts            # TypeScript test suite
```

## Getting started

### Prerequisites

- Rust and the Solana CLI
- Anchor CLI, using the version pinned in `Anchor.toml`
- Node.js and Yarn

### Install

```bash
git clone https://github.com/Armaansaxena/Solana-amm-instruction-introspection
cd Solana-amm-instruction-introspection
npm install
```

### Run the tests

```bash
anchor test
```

## Known limitations

- Not audited. This is a learning project.
- Constant-product pricing only: no oracle, no concentrated liquidity, no multi-hop routing.

## Related

The basic version of this AMM (without introspection): https://github.com/Armaansaxena/Constant-Product-AMM

## Author

Armaan Saxena. [GitHub](https://github.com/Armaansaxena)
