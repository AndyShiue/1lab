<!--
```agda
open import 1Lab.Prelude

open import Algebra.Group.Homotopy.BAut

open import Data.Fin.Properties
open import Data.Fin.Closure
open import Data.Fin.Base
open import Data.Nat.Base
open import Data.Dec
open import Data.Sum
```
-->

```agda
module Data.Fin.Finite where
```

# Finite types

This module pieces together a couple of pre-existing constructions: In
terms of the [[standard finite sets|standard finite set]] (which are
defined for natural numbers $n$) and [deloopings of automorphism
groups], we construct the type of finite types. [By univalence], the
space of finite types classifies maps with finite fibres.

[deloopings of automorphism groups]: Algebra.Group.Homotopy.BAut.html
[By univalence]: 1Lab.Univalence.html#object-classifiers

But what does it mean for a type to be finite? A naïve first approach is
to define "$X$ is finite" to mean "$X$ is equipped with $n : \NN$ and $f
: [n] \simeq X$" but this turns out to be _too strong_: This doesn't
just equip the type $X$ with a cardinality, but also with a choice of
total order. Additionally, defined like this, the type of finite types
_is a set_!

```agda
naïve-fin-is-set : is-set (Σ[ X ∈ Type ] Σ[ n ∈ Nat ] Fin n ≃ X)
naïve-fin-is-set = is-hlevel≃ 2 Σ-swap₂ $
  Σ-is-hlevel 2 (hlevel 2) λ x → is-prop→is-hlevel-suc {n = 1} $
    is-contr→is-prop $ Equiv-is-contr (Fin x)
```

That's because, as the proof above shows, it's equivalent to the type of
natural numbers: The type
$$
\sum_{X : \ty} \sum_{n : \NN}\ [n] \simeq X
$$
is equivalent to the type
$$
\sum_{n : \NN} \sum_{X : \ty} [n] \simeq X\text{,}
$$
and univalence says (rather directly) that the sum of $[n] \simeq X$ as
$X$ ranges over a universe is contractible, so we're left with the type
of natural numbers.

This simply won't do: we want the type of finite sets to be equivalent
to the (core of the) _category_ of finite sets, where the automorphism
group of $n$ has $n!$ elements, not exactly one element. What we do is
appeal to a basic intuition: A groupoid is the sum over its connected
components, and we have representatives for every connected component
(given by the standard finite sets):

```agda
Fin-type : Type (lsuc lzero)
Fin-type = Σ[ n ∈ Nat ] BAut (Fin n)

Fin-type-is-groupoid : is-hlevel Fin-type 3
Fin-type-is-groupoid = Σ-is-hlevel 3 (hlevel 3) λ _ →
  BAut-is-hlevel (Fin _) 2 (hlevel 2)
```

:::{.definition #finite alias="finite-type finite-set"}
Informed by this, we now express the correct definition of "being
finite", namely, being [[merely]] equivalent to some standard finite
set.  Rather than using Σ types for this, we can set up typeclass
machinery for automatically deriving boring instances of finiteness,
i.e. those that follow directly from the closure properties.
:::

```agda
record Finite {ℓ} (T : Type ℓ) : Type ℓ where
  constructor fin
  field
    {cardinality} : Nat
    enumeration   : ∥ T ≃ Fin cardinality ∥
```

<!--
```agda
  Finite→is-set : is-set T
  Finite→is-set =
    ∥-∥-rec (is-hlevel-is-prop 2) (λ e → is-hlevel≃ 2 e (hlevel 2)) enumeration

  instance
    Finite→H-Level : H-Level T 2
    Finite→H-Level = basic-instance 2 Finite→is-set

open Finite ⦃ ... ⦄ using (cardinality; enumeration) public
open Finite using (Finite→is-set) public

instance
  H-Level-Finite : ∀ {ℓ} {A : Type ℓ} {n : Nat} → H-Level (Finite A) (suc n)
  H-Level-Finite = prop-instance {T = Finite _} λ where
    x y i .Finite.cardinality → ∥-∥-proj
      ⦇ Fin-injective (⦇ ⦇ x .enumeration e⁻¹ ⦈ ∙e y .enumeration ⦈) ⦈
      i
    x y i .Finite.enumeration → is-prop→pathp
      {B = λ i → ∥ _ ≃ Fin (∥-∥-proj ⦇ Fin-injective (⦇ ⦇ x .enumeration e⁻¹ ⦈ ∙e y .enumeration ⦈) ⦈ i) ∥}
      (λ _ → squash)
      (x .enumeration) (y .enumeration) i

Finite→Discrete : ∀ {ℓ} {A : Type ℓ} → ⦃ Finite A ⦄ → Discrete A
Finite→Discrete {A = A} ⦃ f ⦄ x y = ∥-∥-rec! go (f .enumeration) where
  open Finite f using (Finite→H-Level)
  go : A ≃ Fin (f .cardinality) → Dec (x ≡ y)
  go e with Discrete-Fin (Equiv.to e x) (Equiv.to e y)
  ... | yes p = yes (Equiv.injective e p)
  ... | no ¬p = no λ p → ¬p (ap (e .fst) p)

Dec→Finite : ∀ {ℓ} {A : Type ℓ} → is-prop A → Dec A → Finite A
Dec→Finite ap d with d
... | yes p = fin (inc (is-contr→≃ (is-prop∙→is-contr ap p) Finite-one-is-contr))
... | no ¬p = fin (inc (is-empty→≃⊥ ¬p ∙e Finite-zero-is-initial e⁻¹))

Discrete→Finite≡ : ∀ {ℓ} {A : Type ℓ} → Discrete A → {x y : A} → Finite (x ≡ y)
Discrete→Finite≡ d = Dec→Finite (Discrete→is-set d _ _) (d _ _)

Finite-choice
  : ∀ {ℓ ℓ'} {A : Type ℓ} {B : A → Type ℓ'}
  → ⦃ Finite A ⦄
  → (∀ x → ∥ B x ∥) → ∥ (∀ x → B x) ∥
Finite-choice {B = B} ⦃ fin {sz} e ⦄ k = do
  e ← e
  choose ← finite-choice sz λ x → k (equiv→inverse (e .snd) x)
  pure $ λ x → subst B (equiv→unit (e .snd) x) (choose (e .fst x))

Finite-≃ : ∀ {ℓ ℓ'} {A : Type ℓ} {B : Type ℓ'} → ⦃ Finite A ⦄ → A ≃ B → Finite B
Finite-≃ ⦃ fin {n} e ⦄ e' = fin (∥-∥-map (e' e⁻¹ ∙e_) e)

private variable
  ℓ : Level
  A B : Type ℓ
  P Q : A → Type ℓ
```
-->

```agda
instance
  Finite-Fin : ∀ {n} → Finite (Fin n)
  Finite-⊎ : ⦃ Finite A ⦄ → ⦃ Finite B ⦄ → Finite (A ⊎ B)

  Finite-Σ
    : {P : A → Type ℓ} → ⦃ Finite A ⦄ → ⦃ ∀ {x} → Finite (P x) ⦄ → Finite (Σ A P)
  Finite-Π
    : {P : A → Type ℓ} → ⦃ Finite A ⦄ → ⦃ ∀ {x} → Finite (P x) ⦄ → Finite (∀ x → P x)

  Finite-⊥ : Finite ⊥
  Finite-⊤ : Finite ⊤
  Finite-Bool : Finite Bool

  Finite-PathP
    : ∀ {A : I → Type ℓ} ⦃ s : Finite (A i1) ⦄ {x y}
    → Finite (PathP A x y)

  Finite-Lift : ∀ {ℓ} → ⦃ Finite A ⦄ → Finite (Lift ℓ A)
```

<!--
```agda
private
  finite-pi-fin
    : ∀ {ℓ'} n {B : Fin n → Type ℓ'}
    → (∀ x → Finite (B x))
    → Finite ((x : Fin n) → B x)
  finite-pi-fin zero fam = fin {cardinality = 1} $ pure $ Iso→Equiv λ where
    .fst x → fzero
    .snd .is-iso.inv x ()
    .snd .is-iso.rinv fzero → refl
    .snd .is-iso.linv x → funext λ { () }

  finite-pi-fin (suc sz) {B} fam = ∥-∥-proj $ do
    e ← finite-choice (suc sz) λ x → fam x .enumeration
    let rest = finite-pi-fin sz (λ x → fam (fsuc x))
    cont ← rest .Finite.enumeration
    let
      work = Fin-suc-universal {n = sz} {A = B}
        ∙e Σ-ap (e fzero) (λ x → cont)
        ∙e Finite-sum λ _ → rest .Finite.cardinality
    pure $ fin $ pure work

Finite-Fin = fin (inc (_ , id-equiv))

Finite-⊎ {A = A} {B = B} = fin $ do
  aeq ← enumeration {T = A}
  beq ← enumeration {T = B}
  pure (⊎-ap aeq beq ∙e Finite-coproduct)

Finite-Π {A = A} {P = P} ⦃ fin {sz} en ⦄ ⦃ fam ⦄ = ∥-∥-proj $ do
  eqv ← en
  let count = finite-pi-fin sz λ x → fam {equiv→inverse (eqv .snd) x}
  eqv' ← count .Finite.enumeration
  pure $ fin $ pure $ Π-dom≃ (eqv e⁻¹) ∙e eqv'

Finite-Σ {A = A} {P = P} ⦃ afin ⦄ ⦃ fam ⦄ = ∥-∥-proj $ do
  aeq ← afin .Finite.enumeration
  let
    module aeq = Equiv aeq
    bc : (x : Fin (afin .Finite.cardinality)) → Nat
    bc x = fam {aeq.from x} .Finite.cardinality

    fs : (Σ _ λ x → Fin (bc x)) ≃ Fin (sum (afin .Finite.cardinality) bc)
    fs = Finite-sum bc
    work = do
      t ← Finite-choice λ x → fam {x} .Finite.enumeration
      pure $ Σ-ap aeq λ x → t x
          ∙e (_ , cast-is-equiv (ap (λ e → fam {e} .cardinality)
                    (sym (aeq.η x))))
  pure $ fin ⦇ work ∙e pure fs ⦈

Finite-⊥ = fin (inc (Finite-zero-is-initial e⁻¹))
Finite-⊤ = fin (inc (is-contr→≃⊤ Finite-one-is-contr e⁻¹))
Finite-Bool = fin (inc (Iso→Equiv enum)) where
  enum : Iso Bool (Fin 2)
  enum .fst false = 0
  enum .fst true = 1
  enum .snd .is-iso.inv fzero = false
  enum .snd .is-iso.inv (fsuc fzero) = true
  enum .snd .is-iso.rinv fzero = refl
  enum .snd .is-iso.rinv (fsuc fzero) = refl
  enum .snd .is-iso.linv true = refl
  enum .snd .is-iso.linv false = refl

Finite-PathP = subst Finite (sym (PathP≡Path _ _ _)) (Discrete→Finite≡ Finite→Discrete)

Finite-Lift = Finite-≃ (Lift-≃ e⁻¹)
```
-->
