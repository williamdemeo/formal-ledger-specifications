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


instance
  Computational-UTXO : Computational _⊢_⇀⦇_,UTXO⦈_ String
  Computational-UTXO = MkComputational computeProof completeness
    where
      open Computational Computational-UTXOS renaming  ( computeProof to computeProof-UTXOS
                                                       ; completeness to completeness-UTXOS)

      module _ {Γ : UTxOEnv}{s : UTxOState} {txTop : TopLevelTx} where

        --------------------------------------------------------------------------
        -- computeProof for UTXO
        --------------------------------------------------------------------------
        computeProof : ComputationResult String (∃[ s' ] Γ ⊢ s ⇀⦇ txTop ,UTXO⦈ s')
        computeProof with computeProof-UTXOS Γ tt txTop
        ... | failure es = failure es
        ... | success (tt , utxosProof) with ¿ UTXO-Premises Γ txTop (UTxOOf s) ¿
        ... | no ¬prem = failure (genErrors ¬prem)
        ... | yes prem with IsValidFlagOf txTop
        ... | true  = success ( ⟦ (UTxOOf s ∣ SpendInputsOf txTop ᶜ) ∪ˡ outs txTop , FeesOf s + TxFeesOf txTop , DonationsOf s + DonationsOf txTop ⟧
                              , UTXO-valid (refl , utxosProof , prem) )
        ... | false = success ( ⟦ (UTxOOf s ∣ CollateralInputsOf txTop ᶜ) , FeesOf s + cbalance (UTxOOf s ∣ CollateralInputsOf txTop) , DonationsOf s ⟧ᵘ
                              , UTXO-invalid (refl , utxosProof , prem) )

        --------------------------------------------------------------------------
        -- completeness for UTXO
        --------------------------------------------------------------------------
        completeness : ∀ s' → Γ ⊢ s ⇀⦇ txTop ,UTXO⦈ s' → map proj₁ computeProof ≡ success s'

        completeness s' (UTXO-valid (refl , utxosProof , prem))
          with computeProof-UTXOS Γ tt txTop in eqU
        ... | failure es = ⊥-elim $ case trans (sym (map-failure {f = proj₁} eqU))
                                               (completeness-UTXOS Γ tt txTop tt utxosProof)
                                    of λ ()
        ... | success (tt , _) with ¿ UTXO-Premises Γ txTop (UTxOOf s) ¿
        ... | no ¬prem = ⊥-elim (¬prem prem)
        ... | yes _ with IsValidFlagOf txTop
        ... | true = refl
        -- ... | ()   -- unreachable because refl fixed isValid = true above

        completeness Γ s txTop s' (UTXO-invalid (refl , utxosProof , prem))
          with computeProof-UTXOS Γ tt txTop in eqU
        ... | failure es = ⊥-elim $ case trans (sym (map-failure {f = proj₁} eqU))
                                               (completeness-UTXOS Γ tt txTop tt utxosProof)
                                    of λ ()
        ... | success (tt , _) with ¿ UTXO-Premises Γ txTop (UTxOOf s) ¿
        ... | no ¬prem = ⊥-elim (¬prem prem)
        ... | yes _ with IsValidFlagOf txTop
        ... | false = refl
        -- ... | true ()   -- unreachable because refl fixed isValid = false above

```
