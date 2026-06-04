# Scoping: actually lifting self-referential associated types (#1260 / #1078)

Status: exploration / design notes. Lives on the experimental fork branch
`experiment/lift-self-referential-assoc-types`.

## Background

`--lift-associated-types` (`charon/src/transform/normalize/expand_associated_types.rs`)
turns trait associated types into type parameters, e.g. `trait Iterator { type Item; }`
becomes `trait Iterator<Item> {}`. The module doc (top of that file) already describes a
fallback for *recursive* cases: when a trait can't be fully lifted, keep its associated
types and add **new associated types** for the otherwise-unfillable paths:

```rust
trait Foo { type FooTy: Foo + Bar; }
// becomes
trait Foo {
    type FooTy: Foo + Bar<Self::FooTy_BarTy>;
    type FooTy_BarTy;   // generated
}
```

That fallback is selected per-trait by `add_type_params = !is_self_referential` in
`compute_trait_modifications` (lines ~768–789). `is_self_referential` is set when the
`CycleDetector` (`src/common.rs`) re-enters a trait that is still `Processing`.

PR #1261 (merged into this experiment as the base) already turned the **panic** from #1260
into a graceful diagnostic by making `TraitRefPath::on_real_tref` fallible. This document
scopes *actually lifting* the pattern instead of erroring.

## What the two issues actually are

- **#1078** — wrong *output*, no panic. A default associated type that refers to itself
  (`type Item = &'a (T, Self::X)`) produces an impl whose `parent_clauseN` refers to
  itself. This is a correctness bug on the **impl / kept-assoc-type side**.
- **#1260** — was a panic (now a diagnostic). An associated-type *bound* creates a cycle
  through a chain of supertraits: `Ring::PrimeSubfield: PrimeField`, and
  `PrimeField: Field: Ring`.

They share the recursive-lifting machinery but fail in different places.

## Empirical findings (this repo, debug build, `--lift-associated-types='*'`)

Repro shape:
```rust
pub trait Ring { type PrimeSubfield: PrimeField; const ZERO: Self; }
pub trait Field: Ring { type Packing; }
pub trait PrimeField: Field {}
pub fn f<R: Ring>() -> R::PrimeSubfield { R::PrimeSubfield::ZERO }
```

1. **The trait declarations transform correctly.** With or without the consuming
   function, the *trait decls* come out as:
   ```text
   trait Ring<Self> { type PrimeSubfield; type Self_Clause2_Clause1_Packing; }
   trait Field<Self, Self_Packing>
   trait PrimeField<Self, Self_Clause1_Packing>
   ```
   `Ring` is correctly detected as self-referential (`add_type_params = false`): it keeps
   `PrimeSubfield` and gains a generated assoc type `Self_Clause2_Clause1_Packing` that
   stands for `<Self::PrimeSubfield as Field>::Packing` (reached through the cycle).
   `Field`/`PrimeField` are lifted normally.

2. **Traits-only crates already lift cleanly — no error.** Removing the function makes the
   diagnostic disappear entirely.

3. **The failure is on the consumer side.** Only when an item *uses* the cycle's traits
   (here the function's `R: Ring` clause + `R::PrimeSubfield::ZERO` body) do we get
   `Could not compute the value of … Field<…>::Packing` / `… PrimeField<…>`. The apply
   phase needs `<R::PrimeSubfield as Field>::Packing` to fill `Field`'s lifted `Packing`
   param, and the correct value is `R`'s generated assoc type `Self_Clause2_Clause1_Packing`
   — but the lookup doesn't make that connection.

4. **The exact set of unresolved paths is order-dependent.** Reordering the trait
   declarations changes which projections fail (variant A vs B), confirming the resolution
   gap interacts with traversal/numbering rather than being a single fixed missing case.

## Root cause (refined)

`UpdateItemBody::lookup_path_on_trait_ref` (lines ~939–1002) resolves, for a use-site
`TraitRef`, the value of a required lifted parameter by walking the trait-ref structure. It
handles `TraitImpl`, `Clause`/`SelfId`, `ParentClause`, `BuiltinOrAuto`, and `Dyn`, but:

- `TraitRefKind::ItemClause(..) => None` (line ~964) — it gives up on item clauses, i.e.
  clauses attached to an associated type. The cyclic case routes precisely through these
  (`Ring::PrimeSubfield`'s bound), so the value can't be found.
- When a required param of a *lifted* trait (`Field::Packing`) is requested **on a kept
  assoc type** of a *self-referential* trait (`Ring::PrimeSubfield`), the value lives in the
  self-referential trait's **generated assoc type** (`Ring::Self_Clause2_Clause1_Packing`).
  Nothing currently bridges "lifted-param request on `Self::PrimeSubfield`" →
  "`Self::<generated assoc type>`".

In short: the **producer** side (trait decls + the generated assoc types) is essentially
correct; the **consumer** side (`lookup_path_on_trait_ref` / `update_generics`) doesn't know
how to read those generated assoc types back out when a cycle trait is used.

Secondary structural limitation: `TraitRefPath` (base + `parent_path: Vec<TraitClauseId>`)
cannot express a step *through an associated type's own clause* (an item clause). Several
code paths therefore can only represent supertrait hops, not "the bound on `Self::Assoc`".

## Design options

### Option 1 — Make the consumer side resolve generated assoc types (recommended first step)
Teach `lookup_path_on_trait_ref` (and the `ItemModifications` for self-referential traits) to
map a lifted-param path requested on a kept assoc type to the generating trait's new
associated type. Concretely: when `compute_trait_modifications` records that
`Ring` keeps `PrimeSubfield` and creates `Self_Clause2_Clause1_Packing`, also record a
*replacement entry* so that the path `<Self::PrimeSubfield as Field>::Packing` resolves to
`Self::Self_Clause2_Clause1_Packing`. Then handle the `ItemClause` arm instead of returning
`None`.
- Pros: matches the already-correct producer output; localized to the apply phase; no AST
  type changes.
- Cons: requires `TraitRefPath`/`AssocTypePath` to express an item-clause step, or a parallel
  lookup keyed differently; fiddly De Bruijn / binder bookkeeping.
- Effort: medium. Risk: medium (touches the hot resolution path; needs broad snapshot review).

### Option 2 — Extend `TraitRefPath` with item-clause steps
Add an explicit "item clause" hop to the path representation so paths can name
`<Self::Assoc as Trait>::T` structurally, then thread it through `on_tref`, `on_real_tref`,
`to_path`, the trie (`TypeConstraintSet`), and `to_name`.
- Pros: the clean, general fix; makes #1078 and #1260 expressible with one mechanism.
- Cons: large, invasive change touching every path manipulation and the constraint trie;
  high snapshot churn.
- Effort: large. Risk: high.

### Option 3 — Detect strongly-connected components and keep all of them
Replace the single-anchor `CycleDetector` flagging with SCC computation so every trait in a
cycle is treated consistently (all `add_type_params = false`). This stabilizes which traits
keep assoc types regardless of traversal order.
- Pros: removes order-dependence; conceptually clean.
- Cons: by itself does **not** fix the consumer-side resolution (the real failure here);
  best combined with Option 1.
- Effort: medium. Risk: medium.

## Recommended path

1. Land #1261 (panic → diagnostic) as the safety net (done; PR open).
2. Prototype **Option 1** on this branch: get the consumer side to read the generated assoc
   types, validated against the #1260 repro *with* the function. This is where the real value
   is and the producer output is already correct.
3. If the path-representation friction in Option 1 proves too sharp, escalate to **Option 2**
   for `TraitRefPath`, then revisit #1078 (which needs the same expressiveness on the impl
   side).
4. Consider **Option 3** as a hardening pass once correctness is in place.

## Concrete experiments / instrumentation to run on this branch

- Add `trace!` in `lookup_path_on_trait_ref` (and the `ItemClause` arm) to print the path,
  the trait-ref kind, and the available `type_replacements` when it returns `None` — confirm
  the missing entry is exactly `<Self::PrimeSubfield as Field>::Packing`.
- Dump `ItemModifications` for `Ring` to confirm the generated assoc type and whether a
  replacement entry mapping the cyclic path → `Self_Clause2_Clause1_Packing` exists (it
  currently does not on the consumer side).
- Build matrix: traits-only (passes today) vs with-consumer (fails); A/B declaration orders;
  with/without the `const ZERO: Self` (the body uses it, exercising the const path).

## Test cases to add (once lifting works)

- The #1260 repro **with** the consuming function (currently `known-failure`
  `issue-1260-self-referential-assoc-ty.rs`) should flip to a passing snapshot showing the
  generated assoc type carried through `f`.
- An indirect-cycle variant with a method that returns the cyclic assoc type.
- The #1078 default-assoc-type case, asserting no self-referential `parent_clauseN`.
- Reordered-declaration variants, asserting identical output regardless of order.
