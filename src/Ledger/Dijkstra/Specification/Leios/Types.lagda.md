---
source_branch: master
source_path: src/Ledger/Dijkstra/Specification/Leios/Types.lagda.md
---

# Leios Primitive Types {#sec:leios-primitive-types}

Leios adds four objects to the chain's traffic: the endorser block, the header
announcement that names it, the committee's votes, and the certificate that
aggregates a quorum of them.
`Ledger.Dijkstra.Specification.Leios.Abstract`{.AgdaModule} recounts the
protocol flow and provides the hash and signature carriers; this module gives
each object its spec-level type and maps it to its wire-format counterpart in
the CDDL of CIP-164's Appendix B[^1].  Wire-level integer widths (the `uint16`
of a reference size, the `uint32` of an announcement size) relax to `ℕ`.  The
module shares `Abstract`{.AgdaModule}'s parameters: the era's epoch structure,
for the slot type, and a `LeiosAbstract`{.AgdaRecord}.

<!--
```agda
{-# OPTIONS --safe #-}

open import Ledger.Prelude
open import Ledger.Core.Specification.Epoch
open import Ledger.Dijkstra.Specification.Leios.Abstract

module Ledger.Dijkstra.Specification.Leios.Types
  (es : _) (open EpochStructure es using (Slot; DecEq-Slot))
  (la : LeiosAbstract es) (open LeiosAbstract la)
  where
```
-->

*The endorser block*
```agda
record EndorserBlock : Type where
  field
    ebTxRefs : List (TxRefHash × ℕ)
```

An endorser block packages the reference structure that
`hashEBRefs`{.AgdaField} identifies — the ordered transaction references whose
shape and rationale are with that field in `Abstract`{.AgdaModule}.  Its CDDL
counterpart is `endorser_block`, whose single entry holds the references as an
`omap<hash32, uint16>`: insertion-ordered, keys unique.

Of the `omap`'s two constraints, the list type keeps the ordering and drops
key uniqueness: duplicate-freedom of the reference hashes is a validity
condition on the announced block, stated beside the nonemptiness and size
conditions of the same protocol step, not a proof field of the record.  The
rules must be able to mention a malformed object in order to reject it: a
duplicate-referencing EB is a thing a peer can send, and its rejection is a
predicate that fails, not a type with no inhabitant.  A proof field would also
forfeit the derived equality below: uniqueness proofs contain negations, and
equality of functions is not decidable.

*The endorser-block identifier*
```agda
hashEB : EndorserBlock → EBHash
hashEB eb = hashEBRefs (EndorserBlock.ebTxRefs eb)
```

`hashEB`{.AgdaFunction} fixes the identity the CIP assigns the `announced_eb`
header field, "computed from the complete EB structure"; what it deliberately
does not fix is the preimage, since `hashEBRefs`{.AgdaField} is abstract and
no byte-exact serialization of the reference list is pinned.  This boundary is
a known conformance cliff: an implementation can agree with the spec on every
rule yet disagree on which endorser block an identifier denotes, so pinning
the preimage is a named prerequisite for conformance testing.  Cardano has
precedent in the block-body hash, whose segmented preimage exists only in
implementation internals.

*The announcement*
```agda
Announcement : Type
Announcement = EBHash × ℕ
```

An announcement is the pair a ranking-block header may carry, the announced
EB's identifier and declared byte size (the optional header group
`announced_eb`, `announced_eb_size`).  A wrong declared size invalidates
nothing (the CIP has honest nodes decline to vote instead), so size agreement
belongs to the voters' checks, not to block validity.  The header's third
Leios field, the `certified_eb` bit, has no spec-level counterpart at all: it
flags that the block's own body carries a certificate, which the spec reads
off the body itself, leaving the bit a syncing optimization of the wire
format.

*The vote*
```agda
record Vote : Type where
  field
    vSlot  : Slot
    vEB    : EBHash
    vVoter : ℕ
    vSig   : VotingSig
```

A vote (CDDL: `leios_vote`) names the voting round and its target:
`vSlot`{.AgdaField} is the slot of the block that announced the EB, and the
pair of `vSlot`{.AgdaField} and `vEB`{.AgdaField} is exactly the message
`vSig`{.AgdaField} signs — the `Slot × EBHash` of `isSignedVote`{.AgdaField},
where the slot binding's rationale is recorded.  `vVoter`{.AgdaField}
identifies the caster by seat index into the epoch's committee (the CDDL's
`voter_id`); no eligibility proof accompanies it, because membership is
determined once per epoch from the stake distribution and verified by lookup.

*The certificate*
```agda
record Certificate : Type where
  field
    cSlot    : Slot
    cEB      : EBHash
    cSigners : ℙ ℕ
    cSig     : AggSig
```

A certificate (CDDL: `leios_certificate`) repeats the quorum's common message
in `cSlot`{.AgdaField} and `cEB`{.AgdaField} and aggregates the rest:
`cSig`{.AgdaField} is the single signature `isSignedAgg`{.AgdaField} checks,
and `cSigners`{.AgdaField} is the set of committee seats that signed.  The
wire format packs that set as the `signers` bitfield, ⌈N/8⌉ bytes over a
committee of size N with bit i set exactly when seat i voted; the spec reads
the bitfield back as the finite set of indices it denotes.

<!--
```agda
unquoteDecl DecEq-EndorserBlock = derive-DecEq ((quote EndorserBlock , DecEq-EndorserBlock) ∷ [])
unquoteDecl DecEq-Vote          = derive-DecEq ((quote Vote          , DecEq-Vote)          ∷ [])
unquoteDecl DecEq-Certificate   = derive-DecEq ((quote Certificate   , DecEq-Certificate)   ∷ [])
```
-->

All four types have decidable equality: the records by derivation and
`Announcement`{.AgdaFunction} from its components; decidability of set
equality over the seat indices covers `cSigners`{.AgdaField}.

[^1]: [CIP-164, Appendix B: Wire Format Specifications (CDDL)](https://github.com/cardano-foundation/CIPs/blob/master/CIP-0164/README.md#appendix-b-cddl).
