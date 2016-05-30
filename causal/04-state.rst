=================
History and state
=================

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

Union of events vs pseudo-mutable shared state
==============================================

3-way merge
===========

If it is possible to "undo" operations (e.g. add a line then remove it, add a
member then remove it), then it is impossible to do correct merges without
knowing previous relevant history. For example:

Merging the same set of states {{ab}, {bc}} in both cases, but with different
history:

.. digraph:: merge_history

    rankdir=BT;
    node [style="filled"];

    x1 [fillcolor="#66ff66",label="b"];
    z1 [label="abc"];
    l1 [fillcolor="#6666ff",label="ab"];
    r1 [fillcolor="#6666ff",label="bc"];
    l1 -> z1 [label="-c"];
    r1 -> z1 [label="-a"];
    x1 -> l1 [color="#666666"];
    x1 -> r1 [color="#666666"];

    x0 [fillcolor="#66ff66",label="abc"];
    z0 [label="b"];
    l0 [fillcolor="#6666ff",label="ab"];
    r0 [fillcolor="#6666ff",label="bc"];
    l0 -> z0 [label="+a"];
    r0 -> z0 [label="+c"];
    x0 -> l0 [color="#666666"];
    x0 -> r0 [color="#666666"];

Node labels indicate the state (an unordered set) at the node. Blue nodes are
the input (nodes to be merged), green is the output (merged state), gray arrows
indicate merge-parents (i.e. nodes with >1 parent).

So we need 3-way merge. This can be defined as a primitive by itself, or
sometimes it's easier to define separate `diff` and `apply` operations for your
state data type, and then we have

    3-way-merge(O, A, B) = apply(B, diff(O, A)) = apply(A, diff(O, B))

TODO:

Possible state types... unordered set is the easiest.

Comparison with state- and operational- CRDTs. We preserve context, more secure
than operational CRDTs which don't.

.. _merge-algorithm:

General merge
=============

TODO: describe our requirement/assumption that nodes are not ancestors of each
other. If they are, then lca calculations will detect this and abort.

We assume there is some 3-way merge for our state data type::

    3-way-merge(o: state, a: state, b: state) -> state

If this results in conflicts, then our algorithm will result in conflicts
during its operation that must be resolved externally, either manually or via
CRDT(?TODO). This is outside the scope of our current discussion.

Simple intuitive derivation
---------------------------

Let's try a recursive definition. As with all recursive derivations, first
let's pretend we already have what we want to define::

    merge(X: [node]) -> state

To make our lives easier, let's derive a simpler form first::

    def lca2(a: node, b: node) -> [node]:
      # lowest common ancestors of 2 nodes
      return max({ v in G | v <= a and v <= b })

    def merge2(a: node, b: node) -> state:
      O = lca2(a, b) # calculate the parent node / nodes
      if size(O) == 1: # base case
        o = O[0]
      if size(O) >= 2: # recurse!
        o = merge(O)
      return 3-way-merge(o.state, a.state, b.state)

There, that wasn't so hard. But now how do we turn ``merge2`` into ``merge``?
As in many other areas of computer science, if we have a binary operation, we
can use `fold`_ (sometimes called `reduce`) to apply it to a non-empty list. We
have to do some minor fiddling as well::

    def merge2'(a: node, b: node) -> node:
      s = merge2(a, b)
      return "new fake node" with state = s, parents = {a, b}

    # attempt at defining "merge"
    def merge-fold-recurse(X: [node]) -> state:
      return fold(merge2', X).state

.. _fold: https://en.wikipedia.org/wiki/Fold_%28higher-order_function%29

"new fake node" means to create a temporary node just for the duration of the
merge algorithm, so that ``fold`` has something to work with. In practise, the
real version optimises this out, but the basic idea is the same and you can
look at our source code to see the difference.

But wait, there's another way of applying the ``fold``. Instead of applying it
to ``merge2``, we apply it *inside* ``merge2``, to ``3-way-merge`` which is
also a binary operation (if we fix the ``o`` argument)::

    def lca(X: [node]) -> [node]:
      # lowest common ancestors of n nodes
      return max({ v in G | v <= x for all x in X })

    def merge-recurse-fold(X: [node]) -> state:
      O = lca(X) # calculate the parent node / nodes
      if size(O) == 1: # base case
        o = O[0]
      if size(O) >= 2: # recurse!
        o = merge-recurse-fold(O)
      def 3-way-merge-with-o(a: node, b: node) -> node:
        s = 3-way-merge(o.state, a.state, b.state)
        return "new fake node" with state = s, parents = {a, b}
      return fold(3-way-merge-with-o, X).state

(If you're confused by this, try to compare this with ``merge2``. The "general
idea" is the same; we just turned binary operations on (a, b) into their n-ary
counterparts.)

So which version is correct? Or perhaps they are identical, and both correct?

A more precise derivation
-------------------------

It turns out, ``fold-recurse`` is correct (and what git recursive-merge does),
and ``recurse-fold`` is wrong. To explain why, we'll justify our steps more
with a semi-formal argument. Hopefully it's precise enough to be turned into a
more rigorous proof later, if one wanted to do that.

First, let's define an `operational edge`. This is an edge from a node to its
parent, where the author of the node intended to introduce some state change,
the "operation". So for example: (a) for a node with a single edge whose state
differs from its parent's, that edge is an `operational edge`; (b) for a node
with many parents where the author only performed the merge algorithm and did
nothing else, none of these edges are `operational edges`; (c) for a node v
with many parents where the author also performed extra operations *on top of*
the merge algorithm, we may instead treat this as virtual nodes (v₀ |leftarrow|
v₁) where v₀ points to the actual parents of v as in case (b) and the edge from
v₁ to v₀ is the operational edge as in case (a). (These cases are not used
in the below proof-sketch, but do become useful in an actual rigorous proof.)

Our other definitions:

- \\ is a binary infix operator denoting set subtraction.
- |SquareUnion| is a binary infix operator denoting set *disjoint* union. That
  is, A |SquareUnion| B is equal to A |cup| B but with the extra assertion that
  A |cap| B = |emptyset|.
- anc*(V) = |bigcup| {anc(v) |forall| v |in| V}, the union of ancestors of all
  nodes in set V.

Our assumptions, which we think are reasonable starting points, but perhaps not
fully rigourous or "maximally simple", are:

1. Given node a, its state ``a.state`` "applies exactly once" all operational
   edges in anc(a). We'll abbreviate this as ``a.state`` |cong| anc(a).

2. If a |le| b, then diff(``a.state``, ``b.state``) |cong| B \\ A.

   a. More generally, for all subgraphs A, B and states s, t: if A |subseteq| B
      and s |cong| A and t |cong| B, then diff(s, t) |cong| B \\ A.

   b. Conversely, for all subgraphs A, D and states s, diffs d: if A and D are
      disjoint and s |cong| A and d |cong| D, then apply(s, d) |cong| A
      |SquareUnion| D.

3. For ``merge(X)``, we want to find some t |cong| anc*(X).

Now follows our proof-sketch. Let A = anc(a), B = anc(b). Sometimes we'll also
treat these node-sets as subgraphs (i.e. including the edges between them);
hopefully it's clear from the context. Now, our goal for ``merge2(a, b)`` is to
to find t |cong| A |cup| B, and show that this is the case.

First, by elementary set theory let's note that A |cup| B
= A |SquareUnion| (B \\ A)
= B |SquareUnion| (A \\ B)
= (A |cap| B) |SquareUnion| (B \\ A) |SquareUnion| (A \\ B)

WLOG let's try to find some s |cong| A \\ B. Define o = lca2(a, b), O = anc*(o)
= A |cap| B. (Note here that o is also a node-set.) Let p be some (yet unknown)
state such that p |cong| O. By (1) we also have ``a.state`` |cong| A. Now O
|subseteq| A so by (2.a) and the previous sentences, we have s = diff(p,
``a.state``) |cong| A \\ O = A \\ B.

How do we find p? Well, if o is a singleton set, then by (1) p = ``o[0].state``
satisfies what we need. If not, then by (3) ``merge(o)`` also satisfies what we
need - and o |subset| A, o |subset| B, due to our "anti-chain" assumption, so
this induction step  eventually reaches a base case, and is therefore a valid
step. (Notice that this is exactly how we defined ``merge2`` earlier.)

Now we can find t. By (1) we have ``b.state`` |cong| B, and we just found s
|cong| A \\ B. So by (2.b) we have apply(``b.state``, s) |cong| B |SquareUnion|
(A \\ B) = A |cup| B. In other words, t = 3-way-merge(``p.state``, ``a.state``,
``b.state``) |cong| A |cup| B. ∎

We can extend this to show why ``fold-recurse`` |equiv| ``merge`` is correct,
essentially by redoing the above proof with (A |cup| B) and C in place of A and
B, when trying to merge X = {a, b, c}, and so on. Similar reasoning also tells
us why ``recurse-fold`` is wrong, because when we execute ``O = lca(X)``, this
is analogous to finding O = A |cap| B |cap| C (``O`` |ne| O, but related), then
trying to calculate t |cong| A |cup| B |cup| C by finding diffs |cong| to each
component of the expression O |SquareUnion| (A \\ O) |SquareUnion| (B \\ O)
|SquareUnion| (C \\ O) - but this is clearly not equal to A |cup| B |cup| C.

As a concrete example, see:

.. digraph:: fold_recurse_vs_recurse_fold

    rankdir=BT;
    node [style="filled"];

    o [label="ab"];
    a [label="a"];
    u [fillcolor="#6666ff",label="abu"];
    b [fillcolor="#6666ff",label="ab"];
    v [fillcolor="#6666ff",label="av"];
    m [fillcolor="#66ff66",label=""];

    a -> o [label="-b"];
    u -> o [label="+u"];
    b -> a [label="+b"];
    v -> a [label="+v"];
    m -> u [color="#666666"];
    m -> b [color="#666666"];
    m -> v [color="#666666"];

For the green node trying to merge the blue nodes, ``fold-recurse`` gives the
correct answer {abuv}, but ``recurse-fold`` gives {auv}.

Commutativity and associativity
-------------------------------

``merge2`` is commutative and associative, and therefore ``merge`` doesn't care
about the order of the input list (and so we can pass in a set instead). This
is basically what we wanted and expected.

TODO: prove this...

Summary
-------

Merge algorithms on DAGs

Requirements:

- there is a partial order on state-update events.

- there exists a ternary operator `3-way-merge` on states which obeys

  Symmetric under a fixed first argument
    |forall| o, a, b: 3-way-merge(o, a, b) = 3-way-merge(o, b, a)

  `Idempotent <https://en.wikipedia.org/wiki/Idempotence>`_ under a fixed first argument
    |forall| o, a: 3-way-merge(o, a, a) = a

  This condition is satisfied by another stronger condition (which is sometimes
  more natural to derive): that there exists functions `diff(state, state)` and
  `apply(state, diff)` that obeys:

  TODO

Consequences:

- The merge algorithm is associative. One has to fiddle with the definition a
  little bit, as follows:

  TODO

Comparison vs CRDTs
===================

Our history-based merge has a similar purpose to CRDTs [#crdt]_, but these are
designed with different assumptions, and have different requirements on what
they need. To compare:

- CRDTs do not transmit history, and apply operation and state-update events
  (for op-based and state-based CRDTs, respectively) without knowledge of their
  intended context - i.e. the knowledge of each event's author, at the time
  that they generated that event. They are specifically designed around being
  able to ignore this context, i.e. to "move" these events around arbitrary
  places of recipients' histories.

  By contrast, history-merge specifically requires the opposite - that there is
  an immutable history that everyone has a strong eventual consistent view of,
  with each state-update being immutably associated with a particular event on
  this history. We do this for security (integrity) reasons, as do DVCSs.

- To achieve the `conflict-free` property (and to be able to correctly re-apply
  events under arbitrarily different contexts), CRDTs place extra requirements
  on what valid events and operations/states are. History-merge only requires
  there be a 3-way-merge over the states.

  To be clear: as far as I know, nobody has studied the differences between
  these requirements, but I strongly suspect that the 3-way-merge requirement
  is weaker (easier). For example, history-merge can support arbitrary set
  remove/re-add operations, but there is no CRDT that does this. [#cset]_

- The term `partial order` is used in completely unrelated ways. To explain:

  - In the context of history-merge, the term refers to the partial order on
    the *events*. It (the order itself) is a data structure with real bits
    representing it, constructed during each run of the protocol. There can be
    no other partial order on the same events.

    It's *possible* (we haven't proved or checked it) that the laws we defined
    for 3-way-merge necessarily induce a partial order on the states. However,
    this is not directly relevant to any of our other discussions about our
    history-merge - and that is why we haven't bothered to prove or check it.

  - In a state-based CRDT, the term refers to a partial order on the *states*.
    It (the order itself) has no "real representation" in terms of a data
    structure, but rather is a mathematical concept inherent to the definition
    of that particular CRDT, chosen independently of (and constant with respect
    to) any run of a protocol that uses it. Conceptually and in general, there
    *could* exist other mathematical partial orders over the same states, that
    also satisfy the same laws as required by the CRDT.

- With history-merge, state updates do not have to increase along the states'
  partial order, if one even exists.

- The history-based merge algorithm is indeed "idempotent, commutative and
  associative" in the same way that state-based CRDTs merge algorithms are. To
  be clear: as described, history-merge is a pure function over a given history
  graph and its associated states; whereas the `merge` of state-based CRDTs is
  typically described as an impure procedure that updates the local current
  state. But fundamentally, they are nearly the same concept, the difference
  being that the CRDT `merge` works without a history.

.. [#crdt] See `CRDTs Illustrated <https://youtu.be/9xFfOhasiOE?t=24m>`_ for
    the definitions of both op-based and state-based CRDTs. The rest of the
    video is also a good introduction. See `Wikipedia
    <https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type>`_ for
    more detailed information.

.. [#cset] That is, without tweaking the representation of the unordered set,
    which has other implications such as privacy loss or extra storage cost.
