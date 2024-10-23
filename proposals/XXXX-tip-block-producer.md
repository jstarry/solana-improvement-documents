---
simd: 'XXXX'
title: Tip Block Producer Instruction
authors: Justin Starry (Anza)
category: Standard
type: Core
status: Review
created: 2024-10-19
feature: (fill in with feature tracking issues once accepted)
---

## Summary

This proposal describes a new system program instruction for tipping the block
producer. The instruction will accumulate tips out of the SVM and will therefore
not be inspectable onchain.

## Motivation

Block tips are currently done out of protocol by validators running Jito
MEV-enabled validators but that approach requires splitting tips into multiple
tip collection accounts to avoid single threading transactions that send tips.
The Solana protocol should provide a mechanism to accumulate tips to the block
producer which doesn't require write locking an account for receiving tips.

## New Terminology

NA

## Detailed Design

A new instruction type with enum variant `13u32` will be added to the system
program for tipping the current block producer.

```rust
pub enum SystemInstruction {
    // ..

    /// Tip the current block producer with the specified amount of lamports.
    ///
    /// # Account references
    ///   0. `[WRITE, SIGNER]` Payer account
    TipBlockProducer {
        lamports: u64,
    },
}
```

The new system instruction will require that the payer account used for the tip
is writable, a signer, owned by the system program, and allocated with 0 data.
If the payer account doesn't have sufficient funds for the tip, the instruction
will fail.

Tips will be accumulated
The SVM's lamport balance checks will be modified to 
TODO: lamport balance checks should be updated
TODO: add syscall so that non system accounts can send tips?

## Alternatives Considered

What alternative designs were considered and what pros/cons does this feature
have relative to them?

## Impact

How will the implemented proposal impacts dapp developers, validators, and core contributors?

## Security Considerations

What security implications/considerations come with implementing this feature?
Are there any implementation-specific guidance or pitfalls?

## Drawbacks *(Optional)*

Why should we not do this?

## Backwards Compatibility *(Optional)*

Does the feature introduce any breaking changes? All incompatibilities and
consequences should be listed.
