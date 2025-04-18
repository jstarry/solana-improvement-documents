---
simd: '0232'
title: Validator Collector Accounts
authors: Justin Starry (Anza)
category: Standard
type: Core
status: Review
created: 2025-01-24
feature: (fill in with feature tracking issues once accepted)
---

## Summary

Allow validators to specify custom collector accounts for both inflation rewards
and block fee revenue.

## Motivation

Validators currently collect block fee revenue in their identity hot wallet
account. This means that program derived addresses are unable to be used for fee
collection. By allowing validators to specify a block fee revenue collector
address, they can use onchain programs to customize how their block revenue is
distributed.

Additionally, validators currently receive inflation rewards directly in their
vote accounts which prevents them from pooling all their income directly into
one account or receiving rewards directly in a fee paying account.

## Alternatives Considered

NA

## New Terminology

NA

## Detailed Design

This proposal requires the adoption of both [SIMD-0180] and [SIMD-0185].
SIMD-0180 adjusts the leader schedule algorithm to make it possible to designate
a specific vote account for a given leader slot. SIMD-0185 adds new collector
address fields to the vote account state.

### Validator Block Revenue Distribution

After processing all of the transactions included in a given block, the protocol
MUST collect all transaction fees into the block producer's specified block
revenue collector account.

With adoption of SIMD-0180, the leader schedule will be keyed by vote account
addresses and can be queried for specific slots to lookup the vote account
associated with a given block. The protocol MUST load this vote account's state
from the **beginning of the previous epoch**. This is the same vote account
state used to build the leader schedule for the current epoch. 

After loading vote account state from the previous epoch, check if the state
field `block_revenue_collector` is initialized. If the value is equal to the
default pubkey (`11111111111111111111111111111111`), fall back to using the vote
account's `node_pubkey` field which is always initialized.

Note that the fee-collector constraints defined in [SIMD-0085] still hold. The
designated fee collector must be a system program owned account that is
rent-exempt after receiving collected block fee rewards. If either of these
constraints is violated, the fees collected for that block will be burned. 

[SIMD-0085]: https://github.com/solana-foundation/solana-improvement-documents/pull/85

### Validator Inflation Reward Distribution

During partitioned epoch rewards calculation, the protocol MUST distribute the
validator commissioned inflation rewards to the 

### Vote Program

A new instruction for setting collector accounts will be added to the vote
program with the enum discriminant value of `16u32` little endian encoded in the
first 4 bytes of the instruction data. The following 32 bytes will serialize
the collector account address and the following byte after that will represent
the kind of collector account.

```rust
pub enum VoteInstruction {
    // ..
    UpdateCollectorAccount { // 16u32
        pubkey: Pubkey,
        kind: CollectorKind,
    },
}

#[repr(u8)]
pub enum CollectorKind {
    InflationRewards = 0,
    BlockRevenue,
}
```

## Impact

Validator identity and fee collector accounts no longer need to be the same
account. This opens up the ability to use PDA accounts for fee collection.

## Security Considerations

NA

## Drawbacks *(Optional)*

NA

## Backwards Compatibility *(Optional)*

This change will require the use of a new feature gate which will enable
collecting fees into custom fee collector addresses if specified by a validator.
