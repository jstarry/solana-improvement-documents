---
simd: 'XXXX'
title: Voter Rotation Overlap
authors:
  - Justin Starry
category: Standard
type: Core
status: Draft
created: 2026-10-26
feature: (fill in with feature key and github tracking issues once accepted)
---

## Summary

When a vote account rotates its BLS voting key, accept votes signed by the
previous key for one epoch after the new key takes effect. The previous key is
added to that epoch's rank map as an "alias rank" of the same vote account.
Certificates can then include either the primary rank or its alias, but never
both. Stake is counted once and equivocation rules apply across both keys. No
vote account state, instruction or certificate encoding changes.

## Motivation

Under Alpenglow, a validator's votes are valid only if they are signed by the
single BLS public key that the epoch's rank map holds for its vote account.
Changing that key is a hard cutover at one exact slot:

- `VoterWithBLS` ([SIMD-0387]) records the new authorized voter for epoch
  `N + 2` and immediately overwrites the vote account's only BLS key slot
  ([handler.rs L119-154][handler-set-voter]).
- The rank map for an epoch is built from the epoch stakes snapshot of vote
  accounts, which for epoch `E` is taken at the first slot of epoch `E - 1`
  ([bank.rs L2762][bank-epoch-stakes], [epoch_stakes.rs L68-137][rank-map]).
  So the new key is the only valid key from the first slot of `N + 2`, and the
  old key is invalid from that same slot.

A validator whose old and new keys are loaded in the same process rotates
without a gap, because the voting client picks whichever loaded keypair matches
the rank map ([voting_utils.rs L150-230][generate-vote]). Many real rotations
are not like that:

- Moving vote authority to a new host, HSM or remote signer. The new signer must
  be voting at the first slot of `N + 2`, and the old one must stop at exactly
  that slot.
- Retiring a host after a key compromise or hardware failure, where the new key
  lives on a different machine.
- Any rotation in which the new signer fails to come up on time. The validator
  stops voting entirely until the problem is fixed, and there is no fallback to
  the old key.

Each missed vote costs the validator rewards and removes its stake from
certificates while it is out. With an overlap epoch, the operator can perform
the switch at any point during epoch `N + 2`, instead of at an exact slot fixed
two epochs in advance.

[SIMD-0185] already anticipated this. It keeps the previous epoch's authorized
voter in `authorized_voters` "to accept votes from both the current and previous
authorized voters to make voter transitions smoother". However, it does not
define an acceptance rule, and under Alpenglow the ed25519 authorized voter no
longer signs votes at all. This proposal defines that rule for BLS votes.

## Dependencies

This proposal depends on the following previously accepted proposals:

- **[SIMD-0185]: Vote Account v4**

    Retains the previous epoch's authorized voter and stores the BLS key.

- **[SIMD-0326]: Alpenglow**

    Defines votes, certificates and the per-epoch validator ranks.

- **[SIMD-0357]: Alpenglow Validator Admission Ticket**

    Filters which vote accounts appear in an epoch's rank map.

- **[SIMD-0387]: BLS Pubkey Management in Vote Account**

    Defines how a vote account's BLS key is set and rotated.

This proposal is designed to compose with the following open proposals:

- **[SIMD-0701]: Vote Account v5**

    Keeps a single mandatory `bls_pubkey_compressed` and caps
    `authorized_voters` at 4 entries. This proposal needs neither to change.

- **[SIMD-0674]: Validator Location Registration** and **[SIMD-0675]:
  Geo-Aware Leader Schedule**

    Take a validator's location from the vote account state captured with
    the epoch stakes, as they already do for its node identity. This
    proposal obtains the previous BLS key the same way.

## New Terminology

- **Rank map `R_E`**: the ordered list of `(vote account, node pubkey, BLS
  pubkey, stake)` entries for epoch `E`, indexed by rank, built from the epoch
  stakes for `E`.
- **Primary rank**: a vote account's rank in `R_E` as computed today.
- **Rotation epoch**: an epoch `E` in which a vote account's BLS key in `R_E`
  differs from its BLS key in `R_{E-1}`.
- **Alias rank**: an extra rank appended to `R_E` for a vote account in its
  rotation epoch. It carries the account's BLS key from `R_{E-1}`.

## Detailed Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC 2119] and [RFC 8174].

### Rank map construction

Let `n` be the number of entries in `R_E` built as today. When the feature is
active (see [Activation](#activation)), `R_E` MUST be extended as follows:

1. Primary ranks `0..n` MUST be computed exactly as today. Their order, their
   entries and the duplicate-key filter do not change.
2. For each primary entry, in primary rank order, an alias entry MUST be
   appended at the next rank `n + j` if all of the following hold:
   - the same vote account has an entry in `R_{E-1}`;
   - its BLS key in `R_{E-1}` (`K_prev`) differs from its BLS key in `R_E`;
   - `K_prev` is not the BLS key of any primary entry in `R_E`, nor of any alias
     entry already appended.
3. An alias entry carries the vote account pubkey, the stake from its primary
   entry in `R_E`, the BLS key `K_prev`, and the node pubkey from `R_{E-1}`.
4. Lookups of a rank by vote account or node pubkey MUST keep returning the
   primary rank. Implementations need a separate lookup from primary rank to
   alias rank.

The total number of ranks `n + k`, where `k` is the number of alias entries,
MUST fit in a `u16`. If it would not, alias entries MUST be dropped from the end
until it fits.

`R_{E-1}` is the rank map of the previous epoch's epoch stakes, which banks
already retain (`MAX_LEADER_SCHEDULE_STAKES`). If those epoch stakes are
unavailable, `R_E` MUST NOT contain alias entries.

Alias keys were proof-of-possession verified when they were registered, the
same as every key in `R_{E-1}`, so they may be treated as PoP-verified.

### Vote account state (v4 and v5)

No vote account field, instruction or account size changes, for Vote
Account v4 ([SIMD-0185]) or Vote Account v5 ([SIMD-0701]):

- **The previous BLS key is not stored in the vote account.** `K_prev` is
  read from the epoch stakes for `E - 1`, the vote account state captured
  when that epoch's rank map was computed. The account's
  `bls_pubkey_compressed` (optional in v4, mandatory in v5) remains the only
  stored key. `VoterWithBLS` keeps overwriting it immediately, as it does
  today. That is safe because every consumer already reads the key from the
  epoch stakes snapshot, not from the live account. [SIMD-0674] specifies
  the same rule for `location`.
- **`authorized_voters` and the 4-entry cap are unaffected.** [SIMD-0185]
  already retains entries for `[current_epoch - 1, current_epoch + 2]`, at
  most 4. This proposal adds no entries, so it never triggers the Vote
  Account v5 rule that `Authorize*` MUST fail when adding a voter would
  exceed 4 after purging.
- **The overlap covers BLS consensus votes only.** The previous key and the
  previous ed25519 authorized voter gain no vote program authority.
  Instructions that require the current epoch's authorized voter, such as
  `UpdateLocation` ([SIMD-0674]), `Authorize*` and `VoterWithBLS`, still
  require it during the rotation epoch.
- **Version changes between `E - 1` and `E`** do not matter. For example, a
  v4 account converted to v5 by `UpdateLocation` copies its BLS key, so it
  gets an alias entry only if its key also changed. Under
  `enforce_validator_location_registration` ([SIMD-0674]), an account must
  be in the active set in both epochs to have an alias.

### Votes

A vote for a slot in epoch `E` MAY be signed with the alias key and carry the
alias rank. It MUST be verified against the alias entry's BLS key. Votes for
slots in epoch `E - 1` are verified against `R_{E-1}` only; there is no forward
overlap.

For every rule in [SIMD-0326] that limits which votes a validator may cast, a
vote under an alias rank MUST be attributed to the same validator as its primary
rank. Examples are at most one notarization per slot, and no finalization after
a skip. Any pair of conflicting votes across the two ranks, signed with either
key in either combination, is a conflict by that vote account. Conflict
detection MUST therefore be keyed by vote account, or by primary rank, rather
than by raw rank. In Agave, the consensus pool assumes the sigverifier already
did these checks ([vote_pool.rs L5][vote-pool]).

### Certificates

Certificate encoding is unchanged. A certificate is an aggregate signature plus
a rank bitmap ([certificate.rs L60][certificate]). In a rotation epoch, bitmaps
index ranks `0..n + k`, and the decoder's maximum validator count MUST be
`n + k`.

A certificate MUST be rejected if, for any primary rank `r` with alias rank
`a(r)`, both `r` and `a(r)` are set. This applies within one bitmap, and across
the primary and fallback bitmaps of a base-3 encoded certificate. It extends
the existing rejection of overlapping primary and fallback bitmaps
([cert_verify.rs L38][cert-verify-overlap]). Stake toward a certificate's
threshold counts each vote account once.

Verification then works as today ([cert_verify.rs L71][cert-verify]). The
verifier aggregates the public keys of the set ranks, using `K_prev` for an
alias rank, and checks the aggregate signature. This needs no extra pairings.
The added cost is one G1 point addition per alias rank set. Certificates grow
only when an alias rank is set, by at most `ceil((n+k)/8) - ceil(n/8)` bytes of
bitmap. `k` is normally zero, or a handful per epoch.

Reward certificates and any other structure that indexes validators by rank
MUST apply the same rule: an alias rank credits its vote account, at most once.

### Voting clients

A voting client MUST sign each vote with at most one of its keys. It MUST NOT
sign votes for the same slot under both the primary and the alias rank. A
client holding the primary key SHOULD use it. A client that holds only the
previous key MAY vote under the alias rank during the rotation epoch.

Clients that refuse to vote when their identity does not match the rank map's
node pubkey (Agave's hot-spare check, [voting_utils.rs L201][hot-spare]) MUST
compare against the alias entry's node pubkey when voting under an alias rank.
That node pubkey comes from `R_{E-1}`.

Operators remain responsible for ensuring that two hosts never vote for the
same vote account at the same time. The overlap lets a handoff happen at any
time during the rotation epoch. It does not make running two active signers
safe.

### Node identity

This proposal does not change node identity handling. The leader schedule,
Turbine and QoS for epoch `E` continue to use the primary entry's node pubkey.
If the identity and the voter are rotated together, which is common when the
authorized voter is the identity keypair, then the new host is still the
leader from the first slot of `E`. In that case the overlap only keeps the
validator's votes counting while the old host finishes its handoff.

### Activation

Behavior is gated by a new feature. `R_E` contains alias entries if and only if
the feature is active at the first slot of epoch `E - 1`, the slot at which the
epoch stakes for `E` are computed. Validators MUST accept alias ranks in votes
and certificates only for slots in such epochs.

### Determinism and emergency rotations

Whether a vote account has an alias rank in epoch `E` MUST be decided only from
the epoch stakes for `E - 1` and `E`. Those are fixed by the first slot of
`E - 1`, are saved in bank snapshots, and are never rebuilt. As a result, every
validator verifies votes and certificates for `E` against the same rank map,
whatever fork it is on. Today's rank map has the same property: it is built
from the epoch stakes alone ([epoch_stakes.rs L334-345][rank-map-init]).

A late opt-out, which would cancel an alias after the epoch stakes for `E` are
fixed, is therefore not possible. It would require either changing epoch
stakes that are already fixed, or checking live vote account state during
vote and certificate verification, which would make validity depend on the
fork.

A compatible extension is an opt-out set at rotation time. It would be a flag
written in the same `VoterWithBLS` instruction that sets the new key, so it
lands in the same epoch stakes snapshot as the key itself. The rank map builder
would then skip the alias for that account. This restores today's behavior for
an emergency rotation, with the old key valid until the end of `N + 1`, and it
needs no faster propagation than the rotation itself. It requires a vote
account field, which only Vote Account v5 ([SIMD-0701]) has room for, and a
vote program change. It is left out of this proposal so that the proposal makes
no vote program change.

### Edge Cases

- **New, re-admitted or VAT-filtered accounts:** an account with no entry in
  `R_{E-1}` gets no alias entry, and neither does an account dropped from
  `R_E`.
- **Unchanged key:** no alias entry.
- **Rotation in consecutive epochs:** the alias always carries the `R_{E-1}`
  key only. A rotation in `E + 1` makes the `R_E` key the next alias.
- **Stake changes between `E - 1` and `E`:** the alias uses the epoch `E`
  stake.
- **Key collisions:** `K_prev` equal to another account's current key, or to
  another alias, causes the alias entry to be dropped.
- **Rank overflow:** see the `u16` rule above.

### Validator Components Affected

| Validator Component             | Impact                              |
|---------------------------------|-------------------------------------|
| Transaction Execution (Runtime) | Rank map construction (epoch E-1)   |
| Virtual Machine                 | None                                |
| Block Packing                   | None                                |
| Consensus                       | Vote/cert verification, voting      |
| Gossip                          | None                                |
| Turbine                         | None                                |
| Snapshots                       | None (derived from epoch stakes)    |
| On-Chain Core BPF Programs      | None                                |
| Other (please describe)         | Reward certificate accounting       |

## Alternatives Considered

- **A second key slot per rank with a second bitmap.** Every certificate would
  carry two bitmaps, even though almost no rank uses the second one. The alias
  ranks in this proposal cost nothing when unused.
- **Separate aggregate signatures for previous keys.** This adds a signature
  and a pairing check to any certificate that includes a rotating validator.
- **Storing the previous BLS key in the vote account.** Vote Account v5
  ([SIMD-0701]) has room for it, in the zeroed tail at offset 608: a 48-byte
  key plus an 8-byte effective epoch. The vote program would write it on
  `VoterWithBLS`. This was rejected for three reasons:
  - It duplicates state the epoch stakes already hold, so it adds a second
    source of truth that could disagree with the rank map.
  - It would not work for Vote Account v4, which has no spare field.
  - Consumers would still have to read it from the epoch stakes snapshot,
    not the live account, to be deterministic. That is the [SIMD-0674]
    reasoning for `location`.
- **Overlap in the epoch before the switch (accepting the next key early).**
  This is symmetric but adds nothing. The operator chooses when the new key is
  set, and the backward overlap already removes the exact-slot cutover.
- **Operational guidance only (load both keys on one host).** This already
  works for single-host rotations. It does not help the multi-host cases in the
  Motivation.

## Impact

- **Validators:** voter and BLS key rotations no longer require a cutover at
  an exact slot. A failed rotation degrades to the old key for one epoch
  instead of a full voting outage.
- **Core contributors:** Agave and Firedancer must change rank map
  construction, vote and certificate verification, conflict detection keyed by
  vote account, reward accounting and the voting client's key selection. The
  vote program does not change.
- **Dapp developers:** none.

## Security Considerations

- **Longer validity of a retired key.** A previous key stays valid for one more
  epoch. Today a rotation requested in epoch `N` already leaves the old key
  valid until the end of `N + 1`. With this proposal, it stays valid until the
  end of `N + 2`. For an emergency rotation after a key compromise, this extends
  the attacker's window by one epoch. During that epoch, conflicting votes
  signed with the stolen key are attributed to the honest vote account. See
  [Determinism and emergency rotations](#determinism-and-emergency-rotations)
  for why this window cannot be cut short after the fact, and for the
  rotation-time opt-out that would remove it.
- **Double counting:** prevented by the rule that a certificate never sets both
  a rank and its alias.
- **Stake of a rotating account:** the alias rank carries ordinary stake, with
  no cap and no exclusion from any threshold. Because a certificate sets a rank
  or its alias but never both, the account's stake counts at most once. An
  attacker holding a retired key has the same power as today, when a stolen key
  stays valid until the end of `N + 1`. The overlap only extends that by one
  epoch.
- **Equivocation:** defined across both keys by attributing alias votes to the
  vote account. Any future slashing proposal (e.g. [SIMD-0204]) MUST treat
  conflicting votes under a rank and its alias as equivocation by that account.
- **Rogue-key attacks:** alias keys were PoP-verified at registration.
  Implementations MUST NOT accept an alias key that did not appear in
  `R_{E-1}`.

## Drawbacks

- It adds a small amount of rank-map and verification complexity to every
  client, for an event (key rotation) that is rare per validator.
- It extends a compromised key's validity by one epoch.

## Backwards Compatibility

This is a consensus change and requires a feature gate. Before activation,
rank maps, votes and certificates are unchanged. After activation, validators
that do not implement this proposal would reject certificates that set alias
ranks.

## Conformance

- Test vectors for certificate verification that cover: a valid certificate
  using an alias rank, rejection when both a rank and its alias are set (within
  one bitmap and across base-3 bitmaps), and rejection of alias ranks in a
  non-rotation epoch.
- A localnet ledger that rotates a vote account's voter and BLS key from one
  host to another. The old host should keep voting under the alias rank into
  the rotation epoch and hand off mid-epoch, before and after feature
  activation.

[SIMD-0185]: https://github.com/solana-foundation/solana-improvement-documents/pull/185
[SIMD-0204]: https://github.com/solana-foundation/solana-improvement-documents/pull/204
[SIMD-0326]: https://github.com/solana-foundation/solana-improvement-documents/pull/326
[SIMD-0357]: https://github.com/solana-foundation/solana-improvement-documents/pull/357
[SIMD-0387]: https://github.com/solana-foundation/solana-improvement-documents/pull/387
[SIMD-0674]: https://github.com/solana-foundation/solana-improvement-documents/pull/674
[SIMD-0675]: https://github.com/solana-foundation/solana-improvement-documents/pull/675
[SIMD-0701]: https://github.com/solana-foundation/solana-improvement-documents/pull/701
[RFC 2119]: https://www.ietf.org/rfc/rfc2119.txt
[RFC 8174]: https://www.ietf.org/rfc/rfc8174.txt
[handler-set-voter]: https://github.com/anza-xyz/agave/blob/v4.4.0-rc.1/programs/vote/src/vote_state/handler.rs#L119-L154
[bank-epoch-stakes]: https://github.com/anza-xyz/agave/blob/v4.4.0-rc.1/runtime/src/bank.rs#L2762
[rank-map]: https://github.com/anza-xyz/agave/blob/v4.4.0-rc.1/runtime/src/epoch_stakes.rs#L68-L137
[rank-map-init]: https://github.com/anza-xyz/agave/blob/v4.4.0-rc.1/runtime/src/epoch_stakes.rs#L334-L345
[generate-vote]: https://github.com/anza-xyz/agave/blob/v4.4.0-rc.1/votor/src/voting_utils.rs#L150-L230
[hot-spare]: https://github.com/anza-xyz/agave/blob/v4.4.0-rc.1/votor/src/voting_utils.rs#L201
[vote-pool]: https://github.com/anza-xyz/agave/blob/v4.4.0-rc.1/votor/src/consensus_pool/vote_pool.rs#L5
[certificate]: https://github.com/anza-xyz/agave/blob/v4.4.0-rc.1/votor-messages/src/certificate.rs#L60
[cert-verify]: https://github.com/anza-xyz/agave/blob/v4.4.0-rc.1/bls-cert-verify/src/cert_verify.rs#L71
[cert-verify-overlap]: https://github.com/anza-xyz/agave/blob/v4.4.0-rc.1/bls-cert-verify/src/cert_verify.rs#L38
