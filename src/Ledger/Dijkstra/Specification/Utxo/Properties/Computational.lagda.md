---
source_branch: master
source_path: src/Ledger/Dijkstra/Specification/Utxo/Properties/Computational.lagda.md
---

```agda
{-# OPTIONS --safe #-}

open import Ledger.Dijkstra.Specification.Abstract
open import Ledger.Dijkstra.Specification.Transaction

module Ledger.Dijkstra.Specification.Utxo.Properties.Computational
  (txs : _) (open TransactionStructure txs)
  (abs : AbstractFunctions txs) (open AbstractFunctions abs)
  where

open import Prelude
open import Ledger.Prelude
open import stdlib-meta.Tactic.GenError using (genErrors)

open import Ledger.Dijkstra.Specification.Utxo txs abs
open import Ledger.Dijkstra.Specification.Script.Validation txs abs
open import Relation.Binary.PropositionalEquality
import Data.Sum.Relation.Unary.All as Sum

open PParams

instance
  _ = Functor-ComputationResult

instance
  Computational-SUBUTXO : Computational _⊢_⇀⦇_,SUBUTXO⦈_ String
  Computational-SUBUTXO = record {go} where
    module go where

      computeProof : (Γ : SubUTxOEnv) (s : UTxOState) (txSub : SubLevelTx) → ComputationResult String (Σ UTxOState (_⊢_⇀⦇_,SUBUTXO⦈_ Γ s txSub))
      computeProof Γ s txSub
        with IsTopLevelValidFlagOf Γ | inspect IsTopLevelValidFlagOf Γ
      ... | false | [ refl ]
        with ¿ SpendInputsOf txSub ≢ ∅
             × inInterval (SlotOf Γ) (ValidIntervalOf txSub)
             × MaybeNetworkIdOf txSub ~ just NetworkId ¿
      ... | (yes p) = success (⟦ UTxOOf s , FeesOf s , DonationsOf s ⟧ , SUBUTXO {txSub = txSub} p)
      ... | (no ¬p) = failure (genErrors ¬p)
      computeProof Γ s txSub | true | [ refl ]
        with ¿ SpendInputsOf txSub ≢ ∅
             × inInterval (SlotOf Γ) (ValidIntervalOf txSub)
             × MaybeNetworkIdOf txSub ~ just NetworkId ¿
      ... | (yes p) = success (⟦ (UTxOOf s ∣ SpendInputsOf txSub ᶜ) ∪ˡ outs txSub , FeesOf s , DonationsOf s + DonationsOf txSub ⟧ , SUBUTXO {txSub = txSub} p)
      ... | (no ¬p) = failure (genErrors ¬p)


      completeness : (Γ : SubUTxOEnv) (s : UTxOState) (txSub : SubLevelTx) → ∀ s' → Γ ⊢ s ⇀⦇ txSub ,SUBUTXO⦈ s' → map proj₁ (computeProof Γ s txSub) ≡ success s'
      completeness Γ s txSub s' (SUBUTXO p)
        with IsTopLevelValidFlagOf Γ | inspect IsTopLevelValidFlagOf Γ
      ... | false | [ refl ]
        with ¿ SpendInputsOf txSub ≢ ∅
             × inInterval (SlotOf Γ) (ValidIntervalOf txSub)
             × MaybeNetworkIdOf txSub ~ just NetworkId ¿
      ... |  (yes p) = refl
      ... |  (no ¬p) = ⊥-elim (¬p p)
      completeness Γ s txSub s' (SUBUTXO p) | true  | [ refl ]
        with ¿ SpendInputsOf txSub ≢ ∅
             × inInterval (SlotOf Γ) (ValidIntervalOf txSub)
             × MaybeNetworkIdOf txSub ~ just NetworkId ¿
      ... |  (yes p) = refl
      ... |  (no ¬p) = ⊥-elim (¬p p)


  Computational-UTXOS : Computational _⊢_⇀⦇_,UTXOS⦈_ String
  Computational-UTXOS = MkComputational computeProof completeness
    where
      computeProof : (Γ : UTxOEnv) (_ : ⊤) (txTop : TopLevelTx) → ComputationResult String (Σ ⊤ (Γ ⊢ tt ⇀⦇ txTop ,UTXOS⦈_))
      computeProof Γ _ txTop
        with ¿ evalP2Scripts (allP2ScriptsWithContext Γ txTop) ≡ IsValidFlagOf txTop ¿
      ... | yes p = success (tt , UTXOS p)
      ... | no ¬p = failure (genErrors ¬p)

      completeness : (Γ : UTxOEnv) (_ : ⊤) (txTop : TopLevelTx) (_ : ⊤)
        → Γ ⊢ tt ⇀⦇ txTop ,UTXOS⦈ tt
        → (map proj₁ $ computeProof Γ _ txTop) ≡ success tt
      completeness Γ _ txTop _ (UTXOS p)
        with ¿ evalP2Scripts (allP2ScriptsWithContext Γ txTop) ≡ IsValidFlagOf txTop ¿
      ... | yes p = refl
      ... | no ¬p = ⊥-elim (¬p p)


{--
  Computational-UTXO : Computational _⊢_⇀⦇_,UTXO⦈_ String
  Computational-UTXO = MkComputational computeProof completeness
    where
    open Computational Computational-UTXOS renaming  ( computeProof to computeProof-UTXOS
                                                     ; completeness to completeness-UTXOS)

    --------------------------------------------------------------------------
    -- computeProof for UTXO
    --------------------------------------------------------------------------

    computeProof : (Γ : UTxOEnv) (s : UTxOState) (txTop : TopLevelTx)
      → ComputationResult String (∃[ s' ] Γ ⊢ s ⇀⦇ txTop ,UTXO⦈ s')

    computeProof Γ (⟦ utxo , fees , donations ⟧ᵘ) txTop with (IsValidFlagOf txTop ≟ true)
    ... | (no ¬valid)
      with ¿ UTXO-Premises Γ txTop utxo ¿
    ... | (no ¬p) = failure (genErrors ¬p)
    ... | (yes p)
      with computeProof-UTXOS Γ tt txTop
    ... | failure es = failure es
    ... | success (tt , utxosProof) =
      success (⟦ utxo ∣ (CollateralInputsOf txTop) ᶜ , fees + cbalance (utxo ∣ CollateralInputsOf txTop) , donations ⟧ , UTXO-invalid ¬valid p utxosProof)

    computeProof Γ (⟦ utxo , fees , donations ⟧ᵘ) txTop | (yes valid)
      with ¿ UTXO-Premises Γ txTop utxo ¿
    ... | (no ¬p) = failure (genErrors ¬p)
    ... | (yes p)
      with computeProof-UTXOS Γ tt txTop
    ... | failure es = failure es
    ... | success (tt , utxosProof) =
      success (⟦ (utxo ∣ SpendInputsOf txTop ᶜ) ∪ˡ outs txTop , fees + TxFeesOf txTop , donations + DonationsOf txTop ⟧ , UTXO-valid valid p utxosProof)

    --------------------------------------------------------------------------
    -- completeness (mirrors computeProof)
    --------------------------------------------------------------------------

    completeness : (Γ : UTxOEnv) (s : UTxOState) (txTop : TopLevelTx) (s' : UTxOState)
      → Γ ⊢ s ⇀⦇ txTop ,UTXO⦈ s' → (map proj₁ $ computeProof Γ s txTop) ≡ success s'

    completeness Γ (⟦ utxo , fees , donations ⟧ᵘ) txTop s' x = {!!}
-- --}
```
