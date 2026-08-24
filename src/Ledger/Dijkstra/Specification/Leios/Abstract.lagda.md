---
source_branch: master
source_path: src/Ledger/Dijkstra/Specification/Leios/Abstract.lagda.md
---

# Leios Voting Cryptography {#sec:leios-voting-cryptography}

In Leios[^1], a committee of stake pools votes on an announced endorser block
(EB), and a quorum of votes is aggregated into a certificate that the next
ranking block may carry.  The ledger produces none of this cryptography; it
only verifies it: registration checks a pool's voting key against a proof of
possession, and block validation checks a certificate's aggregate signature
against the voters' keys.  The record `LeiosAbstract`{.AgdaRecord} collects
exactly this verification surface, leaving the concrete signature scheme
abstract in the manner of `PKKScheme`{.AgdaRecord}.

The module is parameterized by the era's epoch structure, of which only the
slot type is used.

<!--
```agda
{-# OPTIONS --safe #-}

open import Ledger.Prelude
open import Ledger.Core.Specification.Epoch

module Ledger.Dijkstra.Specification.Leios.Abstract
  (es : _) (open EpochStructure es using (Slot))
  where
```
-->
*Types*
```agda
record LeiosAbstract : Type₁ where
  field
    VotingKey  : Type
    KeyProof   : Type
    VotingSig  : Type
    AggSig     : Type
    EBHash     : Type
    TxRefHash  : Type
```

A voting key registers together with a `KeyProof`{.AgdaField}, its proof of
possession; each vote carries a `VotingSig`{.AgdaField}, and a certificate
carries a single `AggSig`{.AgdaField} for its whole quorum; an endorser block
is identified by its `EBHash`{.AgdaField}, and each transaction it references
by a `TxRefHash`{.AgdaField}.

*Verification predicates*
```agda
    validKeyProof  : VotingKey → KeyProof → Type
    isSignedVote   : VotingKey → Slot × EBHash → VotingSig → Type
    isSignedAgg    : List VotingKey → Slot × EBHash → AggSig → Type
```

The two signature predicates take the message actually signed: the pair of
the announcing block's slot and the endorser-block hash.  Binding the slot
ensures a vote attests validation of the EB against the very ledger state its
certified closure will extend.  `validKeyProof`{.AgdaField} is the
registration-time check of a key's proof of possession;
`isSignedVote`{.AgdaField} gives the meaning by which consensus filters
individual votes, which never appear on chain; `isSignedAgg`{.AgdaField} is
the on-chain check that a certificate's aggregate signature verifies against
the keys of the voters it lists.

*The endorser-block identifier*
```agda
    hashEBRefs     : List (TxRefHash × ℕ) → EBHash
```

An endorser block is an ordered list of transaction references — pairs of a
reference hash and a declared byte size — and its identifier is the hash of
that reference structure, so the identifier is checkable before any
referenced transaction data arrives.  A reference hash covers the complete
transaction bytes, witnesses included; the ledger's transaction id identifies
only the transaction body, so it could not pin the exact transactions the
voters validated.

<!--
```agda
  field
    ⦃ Dec-validKeyProof ⦄ : validKeyProof ⁇²
    ⦃ Dec-isSignedVote  ⦄ : isSignedVote ⁇³
    ⦃ Dec-isSignedAgg   ⦄ : isSignedAgg ⁇³

    ⦃ DecEq-VotingKey ⦄ : DecEq VotingKey
    ⦃ DecEq-KeyProof  ⦄ : DecEq KeyProof
    ⦃ DecEq-VotingSig ⦄ : DecEq VotingSig
    ⦃ DecEq-AggSig    ⦄ : DecEq AggSig
    ⦃ DecEq-EBHash    ⦄ : DecEq EBHash
    ⦃ DecEq-TxRefHash ⦄ : DecEq TxRefHash
```
-->

All six types have decidable equality and all three predicates are
decidable.  Unlike `PKKScheme`{.AgdaRecord}, the record exposes no signing or
aggregation functions and no correctness law relating them: key generation,
vote signing, and aggregation happen in the node, outside the ledger, so only
verification enters the rules.

CIP-164 instantiates the interface with BLS12-381 in the MinSig
configuration: signatures, the aggregate included, live in the group G₁
(48 bytes compressed) and verification keys in G₂ (96 bytes), the right
trade-off when votes dominate wire traffic while keys travel only at
registration.  A certificate's aggregate signature is the product of the
quorum's vote signatures over the common message, verified against the
signers' keys, and possession is proved by a signature on the key itself,
which closes the rogue-key attack that same-message aggregation would
otherwise admit.  The scheme meets CIP-164's requirements for votes and
certificates (its Appendix A): succinct registration, rotation by
re-registration, small votes, compact certificates, and fast verification.

Nothing here is specific to Leios beyond the message type; Peras certificates
rest on the same aggregate-signature pattern, so the intent is to promote
this interface into `Ledger.Core`{.AgdaModule}, beside
`CryptoStructure`{.AgdaRecord} where both protocols could share it, once its
shape settles under use.  Until then the record stays local to this era.

[^1]:  [CIP-164, "Ouroboros Linear Leios"](https://github.com/cardano-foundation/CIPs/blob/master/CIP-0164/README.md).
