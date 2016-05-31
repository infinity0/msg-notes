======
Proofs
======

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

Causal ordering
===============

.. _diff-apply-model:

Formal model for diff and apply
-------------------------------

This is similar to, but not exactly the same as, darcs' `theory of patches`.
Their theory is more developed with more "useful" results, but is also places
more constraints on what valid patches are.

We try to be more general (less constrained), to demonstrate that the rest of
our work depends on fewer assumptions than their model. For example, we don't
need a notion of "compose two patches into another patch", and we only have
partial notions of "applying n patches is equivalent to <n other patches".

Objects of our investigation:

S
  set of all possible states
D
  set of all possible patches
▵ :: S x S |rightarrow| D
  binary operation: diff an earlier state with a later state; always succeeds
· :: S x D |rightarrow| S |cup| {|bot|}
  binary operation: apply a patch to a state; may fail, represented by |bot|

Axioms:

1. Zero diff

   |forall| s |in| S: s ▵ s = 0

2. Diff-apply conversion

   |forall| s, t |in| S; d |in| D: s · d = t |iff| d = s ▵ t

3. Apply cancellation

   |forall| s |in| S; d, e |in| D: s · d = s · e |ne| |bot| |implies| d = e

   Note the converse is already true due to how = works.

4. Diff inverse

   |forall| s, t |in| S: s ▵ t = -(t ▵ s)

5. Diff cancellation

   |forall| s |in| S; d |in| D: s · d |ne| |bot| |implies| s · d · -d = s

   Note the converse is already true due to the definition of ·.

6. Weak commutativity (optional)

   |forall| o, a, b |in| S: o · (o ▵ a) · (o ▵ b) = o · (o ▵ b) · (o ▵ a)

Lemmas: (TODO)

1. Zero apply (by A2, A1)

   |forall| s |in| S: s · 0 = s

2. Diff collapse (by A2, A3)

   |forall| s |in| S; d |in| D: s ▵ (s · d) = d

3. Apply collapse (by A2, A3)

   |forall| s, t |in| S: s · (s ▵ t) = t

Theorems:

Let 3-way-merge(o, a, b) = b · (o ▵ a)

1. First argument defines the identity element.

   | 3-way-merge(o, a, o) = a (by L3)
   | 3-way-merge(o, o, a) = a (by A1, L1)

2. If A6, then 3-way-merge is commutative under fixed o (by L3, A6)

   |forall| o, a, b |in| S: b · (o ▵ a) = a · (o ▵ b)

Greeting protocols
==================

Key-rotation ratchets
=====================
