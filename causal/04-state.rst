=================
History and state
=================

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

Now back to session semantics. If you're coming into this directly from an
external resource, make sure you've read the first parts of :doc:`01-ordering`.

Regarding membership, so far in our data model we've only defined that each
message is associated with a set of members. Now, suppose we reach a fork in
our session history G, i.e. \|max(G)\| > 1, and the messages in this set all
have different members. What then, is the membership of our current session G?

It's good to be lazy in computer science, so we first start with a rain check -
"do we really need to answer this"? Unfortunately yes, and the result *must* be
a single membership set, and not (e.g.) a set of "possible membership sets": if
the user wanted to send a new message *right now* before the fork is resolved,
we must send it to *some specific* set of members, and we must indicate this
somewhere in the UI so they *know* who they are sending it to.

Union of events vs dynamic state
================================

To recap: we want to develop a dynamic group session protocol, where "dynamic"
means "members may change at any time". Our security constraints around context
and ordering led us to represent the session history as an immutable graph of
messages, whose edges represent a transitively-reduced |le| relationship.

We make the high-level observation that, in models based on an immutable
history of events, each event may be associated with two types of data: with
type 1 the <data> of the whole is just the union of the <data> of every event;
with type 2 the <data> of the whole is an "aggregated form" of the <data> of
every event, and the former "changes" as we accept more events into the local
history. Applying this to group sessions and version control, we have:

+-------------------+-------------------+-------------------------------+
| Model             | Type 1 "union"    | Type 2 "dynamic"              |
+===================+===================+===============================+
| Group session     | Message content   | Session / message membership  |
+-------------------+-------------------+-------------------------------+
| Version control   | Commit message    | Commit tree content           |
+-------------------+-------------------+-------------------------------+

In some sense, the messages (union of events) is the "main focus" of our
system, but membership (dynamic state) is also quite important - and where the
hard problems lie. Luckily, in DVCSs it's the commit tree content (dynamic
state) that is the "main focus" of those systems, and we can use solutions from
there to inspire solutions for the analogous problems in ours.

It's not immediately obvious that there is an "aggregated form" of the tree
content of multiple commits. But in fact, that is what a merge algorithm does:
assuming there are no conflicts, it gives us a single output tree state, for
any input set of forked branch heads.

For group sessions, we don't want to force users to "add a merge commit" (i.e.
write a new message) every time there is a fork. What we *can* do now however,
is to define "the" state of a forked history G, in line with our analogies:

**members(G)** = merge[G](max(G))
    Current session membership at G, i.e. after having received/accepted all
    messages in G, and no other messages.

Now, we need an algorithm merge[G](X) to merge the state of a set of nodes,
like the ones that exist in modern DVCSs. Furthermore, the algorithm must never
require manual conflict resolution - we must have a result *before* the user
writes any message.

From here on, we'll ignore the `type 1` column and message content. When we
refer to the "state" of a node, we mean the `type 2` state, and when we refer
to "merged state", we mean the aggregated state. In our group session system,
`state` represents a set of members, but for now to keep things simple, we'll
ignore this and mostly talk about an opaque semantics-free `state`. To re-state
our aim in these terms, we want to define a function:

**merge[G](X)** -- where:
    - G is a :doc:`partial order <01-ordering>` history graph, where each node
      is associated with some type-2 state (e.g. "set of members")
    - X is a set of nodes |in| G
    - the output is a state

3-way-merge
===========

To begin with, note that it's impossible to do correct merges without knowing
previous relevant history. [#mhst]_ To demonstrate this, see the below cases.
In both cases, we want to merge the same states - unordered sets {a, b} and
{b, c}, but the two cases have different histories:

.. digraph:: merge_history

    rankdir=RL;
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

The node label "bc" means the state at that node is {b, c}, and so on. Edges
are labeled with the operation that the child node performs on the state at the
parent node. Blue nodes are the nodes to be merged, green is the output (a
candidate node with merged state), and gray edges indicate merge parents (i.e.
nodes with >1 parent).

So as a minimum requirement, our merge algorithm must use *some* information
from the previous history. The most basic version of this is a 3-way merge:

3-way-merge(o, a, b) -- where
    - o, a, b are states
    - o is interpreted as some form of both a's past and b's past
    - the output is a state, or |bot| if there is a merge conflict

Based on intuition, we suggest that this should satisfy some invariants. First,
if nothing changed on one branch, then it should simply use the other one, i.e.
|forall| o, a: 3-way-merge(o, o, a) = 3-way-merge(o, a, o) = a. Next, in a
forked history neither side is favoured - so the order of the last arguments
should not change the result, i.e. |forall| o, a, b: 3-way-merge(o, a, b) =
3-way-merge(o, b, a). We'll come back to these later.

To get an intuitive feel on how it should work, we can define 3-way-merge in
terms of more familiar `diff` and `apply` operations:

3-way-merge(o, a, b) = apply(b, diff(o, a)) = apply(a, diff(o, b))

This is easier to understand, but specifying invariants equivalent to the ones
above is more complex; see :ref:`the appendix <diff-apply-model>` for details.
In a real implementation it suffices to define only 3-way-merge, and it's also
slightly more efficient.

From our apply-diff definition, we see a potential source of conflicts. We
apply a diff (o, a) on top of a context (b) that it was not intended for. How
likely this is to happen, depends on the state data type. For example, this can
quite easily happen with lists-of-lines (as DVCSs use), and happens less often
with semantic diff/merge algorithms.

Let's start with the simplest option to represent group session membership, an
unordered set, and try to define 3-way-merge for this data type.

| Set-3-way-merge(o, a, b)
| = Set-apply(b, Set-diff(o, a))
| = Set-apply(b, [insert (a \\ o), delete (o \\ a)])
| = b |cup| (a \\ o) \\ (o \\ a)
| = a |cup| b \\ o |cup| (a |cap| b |cap| o) -- equivalent to the previous;
 more "obviously" symmetric but probably slightly less efficient

It's fairly straightforward to check that this satisfies our invariants above.
To check symmetry with respect to {a, b}, it helps to draw a Venn diagram.

We're in luck - this is well-defined for all arguments, and no conflicts can
result from this algorithm. So let's use this for now as our state data type,
and continue with the rest of the merge algorithm.

.. [#mhst] If the `state` data type does not allow `undo` operations, then we
    can get by without knowing history. This is exactly how state-based CRDTs
    work, see :ref:`below <comparison-vs-crdts>`. In that literature, "no undo"
    is stated instead as "states must increase along some partial order". This
    is unrelated to our partial order on G, also explained below.

    Our system already has history however, so we wanted to start by exploring
    the minimal amount of additional complexity, i.e. to use a simple unordered
    set as our `state` data type for membership sets. This *does* allow `undo`,
    such as adding then removing a member. Similarly, DVCSs that use line-based
    diff algorithms also allow `undo` - you can remove one line from a file,
    commit it, then re-add it in a later commit. Neither unordered sets nor
    line-lists are suitable as a state-based CRDT.

    If these paragraphs confused you, just ignore them and ignore state-based
    CRDTs; it's not essential for understanding the rest of this document.

General merge
=============

What about more complex histories? It turns out, we can build a merge algorithm
that works on arbitrary histories, using only 3-way-merge (which doesn't query
the history) and queries on the history. There are several advantages to this:

It is completely independent of the *state type* - i.e. merge does not need to
manipulate state directly, it simply uses 3-way-merge (which does do that). So
if we change our state type, we only need to provide a new 3-way-merge, and not
a completely new merge algorithm.

Another advantage is that, since it doesn't touch the state directly, it also
does not generate merge conflicts beyond what 3-way-merge generates. We saw
earlier that our 3-way-merge for unordered sets never results in conflicts, so
things are looking pretty good for our original requirements.

We didn't invent this algorithm ourselves: git did the hard work, via years of
mailing list discussions and engineering experience. However, it's unclear if
they know this is a *correct* solution or just an heuristic: even now, ``man
git-merge`` says "[the default merge algorithm] has been reported to result in
fewer merge conflicts without causing mismerges by tests done on actual merge
commits taken from Linux 2.6 kernel development history."

Our novel contribution here then, is a proof sketch that this merge algorithm
is indeed *correct* and *unique*, derived from some fairly simple proposals on
how "reasonable" merge algorithms should behave. We'll get to that later; first
we go through a more "blind" derivation, that's hopefully more intuitive if you
haven't seen how this works before.

To restate our problem, we want to define::

    merge(G: graph, X: {node}) -> state

and we assume there is some 3-way-merge for our state data type::

    3-way-merge(o: state, a: state, b: state) -> state

Notation wise: ``f(x: T0) -> T1`` means that input argument ``x`` has type
``T0`` and ``f`` returns a value of type ``T1``; ``{T}`` is the type of an
immutable, unordered set whose inner values are of type ``T``.

To simplify our pending explanation, we'll assume some things for now, then
un-assume and handle them after we've explained the core of the algorithm.

1.  X is an anti-chain, i.e. no two nodes in it are |le| each other. Later
    we'll describe how to detect and handle this case.

2.  X is a list. Later we'll show that the merge result is the same regardless
    of the order of elements. (Duplicate elements are handled by (1)).

Simple intuitive derivation
---------------------------

Let's try a recursive definition. As with all recursive derivations, first
let's pretend we already have what we want to define::

    merge(G, X: [node]) -> state                # omitting "G: graph" to reduce clutter

To make our lives easier, let's derive a simpler form first::

    def lca2(G, a: node, b: node) -> [node]:
      # lowest common ancestors of 2 nodes
      return max({ v in G | v <= a and v <= b })

    def merge2(G, a: node, b: node) -> state:
      O = lca2(G, a, b)                         # calculate parent node(s), for 3-way-merge
      if size(O) == 1:                          # base case
        s_o = O[0].state
      if size(O) >= 2:                          # recurse!
        s_o = merge(G, O)
      return 3-way-merge(s_o, a.state, b.state)

There, that wasn't so hard. But now how do we turn ``merge2`` into ``merge``?
As in many other areas of computer science, if we have a binary operation, we
can use `fold`_ (sometimes called `reduce`) to apply it to a non-empty list. We
have to do some minor fiddling as well::

    def merge-fold-recurse(G, X: [node]) -> state:
      def merge2'(a: node, b: node) -> node:
        s = merge2(G, a, b)
        return "new temp node" in G with state = s, parents = {a, b}
      return fold(merge2', X).state

.. _fold: https://en.wikipedia.org/wiki/Fold_%28higher-order_function%29

"new temp node" means to create a temporary node in G just for the duration of
``fold``, so that it has something to work with. We'll get rid of this soon
below (in a real implementation G should be immutable anyway) but for now this
gives a good intuition on how the algorithm works.

But wait, there's another way of applying the ``fold``. Instead of applying it
to ``merge2``, we apply it *inside* ``merge2``, to ``3-way-merge`` which is
also a binary operation (if we fix the ``o`` argument)::

    def lca(G, X: [node]) -> [node]:
      # lowest common ancestors of n nodes
      return max({ v in G | v <= x for all x in X })

    def merge-recurse-fold(G, X: [node]) -> state:
      O = lca(G, X)                             # calculate the parent node / nodes
      if size(O) == 1:                          # base case
        s_o = O[0].state
      if size(O) >= 2:                          # recurse!
        s_o = merge-recurse-fold(G, O)
      def 3-way-merge-with-o(a: node, b: node) -> node:
        s = 3-way-merge(s_o, a.state, b.state)
        return "new temp node" in G with state = s, parents = {a, b}
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

2. If a |le| b, then diff(``a.state``, ``b.state``) |cong| anc(b) \\ anc(a).

   a. More generally, for all subgraphs A, B and states s, t: if A |subseteq| B
      and s |cong| A and t |cong| B, then diff(s, t) |cong| B \\ A.

   b. Similarly, for all subgraphs A, D and states s, diffs d: if A and D are
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

To extend this to three arguments, imagine that we redo the proof, but with (A
|cup| B) and C in place of A and B, and with ``merge2(a, b)`` (which we just
proved) in place of ``a.state``. We will find that we need some o such that O =
anc*(o) = (A |cup| B) |cap| C. In our 2-arg proof, this was o = lca2(a, b). In
our 3-arg proof, this must be instead be o = lcaU({a, b}, c), where:

lcaU(A, b)
    | = max({ v |in| G | v |in| anc(b) |and| v |in| anc*(A)})
    | = max({ v |in| G | v |le| b |and| v |le| a for some a |in| A })

Then, the rest of the proof follows exactly as for the 2-arg case. We can also
see that the result is the same regardless of the order of arguments - in all
cases, we are finding some t |cong| anc*({a, b, c}). We can deduce by further
induction, that ``fold-recurse`` |equiv| ``merge`` is correct, and symmetric
with respect to the order of its arguments. ∎

Similar lines of reasoning also tell us why ``recurse-fold`` is wrong, because
when we execute ``O = lca(X)``, this is analogous to finding O = A |cap| B
|cap| C (``O`` |ne| O, but related), then trying to calculate t |cong| A |cup|
B |cup| C by finding diffs |cong| to each component of the expression O
|SquareUnion| (A \\ O) |SquareUnion| (B \\ O) |SquareUnion| (C \\ O) - but this
is clearly not equal to A |cup| B |cup| C.

As a concrete example, see:

.. digraph:: fold_recurse_vs_recurse_fold

    rankdir=RL;
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

.. _merge-algorithm:

Complete algorithm
------------------

Using lcaU from our above proof, we can get rid of our earlier "new temp node"
shenanigans. Sadly, we can no longer use fold and must iterate manually:

.. code-block:: python

    def lcaU(G, A: {node}, b: node) -> {node}:  # note 1
      # lowest common ancestors of a node-set and a node
      return "max({ v in G | v <= b and v <= a for some a in A })"

    def mergeU(G, A: {node}, s_a: state, b: node) -> state:
      if A empty:
        return b.state
      O = lcaU(G, A, b)                         # calculate parent node(s), for 3-way-merge
      CHECK( O & (A | {b}) is empty )           # anti-chain check, see below
      s_o = merge(G, O)                         # recurse upwards, to ancestors
      return 3-way-merge(s_o, s_a, b.state)     # note 2

    def merge_(G, B: {node}, A: {node}, s_a: state) -> state:
      if B empty:
        return s_a
      b = choose_any_element(B)
      s = mergeU(G, A, s_a, b)
      return merge_(G, B - {b}, A | {b}, s)     # tail recurse sideways, to remaining args

    def merge(G, X: {node}) -> state:
      return merge_(G, X, {}, default-state)    # note 3

1. See appendix (TODO: link) for an optimised "real implementation" of this.
2. To reduce clutter, we assume 3-way-merge doesn't give merge conflicts.
   Extending the code to handle that is a straightforward engineering exercise.
3. We could short-cut ``merge`` to check if X is empty or a singleton, but
   these cases are in fact already covered in the child functions.

Now we can check that each input set is an anti-chain. Consider lcaU(A, b) and
suppose we know A is an anti-chain. Then, if a |le| b for any a |in| A, either
a or b will be within the result of ``lcaU``. Equivalently, if lcaU(A, b) |cap|
(A |cup| {b}) = |emptyset|, then we know that b is causally independent of all
a |in| A. If we add this check to ``mergeU``, then ``merge_`` will start with a
base case A = |emptyset| which is an anti-chain, and as it processes elements
from B and moves them into A, it will by induction check all pairs {x, x'} from
X for causal independence.

For our group session, we reject such inputs because they break our protocol
invariants, and CHECK simply throws an exception to be handled further up the
stack. [#tred]_ During run-time, whenever we accept a message m, we must check
that merge(pre(m)) does not throw an exception. This enforces the transitive
reduction property, as promised in the first chapter.

Certain extreme history graphs can cause our recursive ``merge`` to stack
overflow. It is possible to implement ``merge`` iteratively, but the non-tail
recursion from ``mergeU`` back to ``merge`` forces such an implementation to be
very complex and hard to understand. A better solution is to simply add an LRU
cache to ``merge``. This is sufficient to prevent overflow, as long as we run
the aforementioned merge(pre(m)) check for every accepted message.

To summarise, if we fulfill the following requirements:

- We have a partial order G on state-update events (nodes).

- We provide a ternary operator `3-way-merge` on states which obeys: [#idem]_

  Identity under the fixed first argument
    |forall| o, a: 3-way-merge(o, a, o) = 3-way-merge(o, o, a) = a

  Commutative under a fixed first argument
    |forall| o, a, b: 3-way-merge(o, a, b) = 3-way-merge(o, b, a)

  Alternatively, see :ref:`the appendix <diff-apply-model>` for the equivalent
  model stated in terms of `diff` and `apply` operations.

Then we can define `merge` as ``merge`` from the pseudo-code above. Its
properties are:

- commutative - the input is an unordered set.

- associative, in some sense. That is, the following two are equivalent, for
  all node-sets A, B:

  - merge(A |cup| B)
  - merge({v} |cup| B) where v = new node { parents = A, state = merge(A) }

- closed (i.e. free from merge-conflicts), if 3-way-merge is closed

.. [#tred] For other systems, such as DVCSs, this is not such a crucial error;
    they can instead ignore the extraneous older nodes, and run the algorithm
    as if the inputs were max(anc*(X)) - which is definitely an anti-chain.
    Tweaking our algorithm to do this efficiently is left as an exercise, but
    note that it is not enough to let CHECK continue or e.g. ``return as``.

.. [#idem] There is another nice property for 3-way-merge to have, but it's not
    strictly necessary for us:

    `Idempotent <https://en.wikipedia.org/wiki/Idempotence>`_ under a fixed first argument
      |forall| o, a: 3-way-merge(o, a, a) = a

    If this holds, then `merge` is also idempotent (i.e. merge(X) = a, if
    |forall| x |in| X: ``x.state`` = a). We actually do have both of these "by
    accident" because 3-way-merge on unordered sets happens to be idempotent,
    but it's not a *necessary* property. It has no effect on our derivation of
    the general merge algorithm, nor any other results here.

    :ref:`State-based CRDTs <comparison-vs-crdts>` *do* require this, because
    their systems can't deduplicate identical events received more than once.
    We *can* do this - it's necessary, to protect against replay attacks.

Other topics
============

Avoiding merges
---------------

Others have suggested instead to avoid the complexities with merge algorithms,
and just *define* forked histories as completely different continuations of the
session, never to be reconciled via a merge.

However, this misunderstands the semantics of forks - and does not actually
solve the original problem. Forks represent not a conscious user decision to
"create a new thread of discussion", but express an inherent property of our
universe that two events may happen at the same time. Even if we supported
user-conscious forks, we would *still* have to support these "accidental"
forks: two users that *don't want to consciously fork* (i.e. *want* the session
to proceed as a *single branch*) could *still* create an accidental fork. This
possibility also exists with DVCSs - for example, two users can end up with
forked histories without ever having explicitly run ``git branch``.

If we avoid merges, we in effect assert that different scenarios (accidental vs
conscious forks) are equivalent. This prevents the user from expressing their
intentions accurately, and we consider this an unacceptable design choice.

.. _comparison-vs-crdts:

Comparison vs CRDTs
-------------------

Our history-based merge has a similar purpose to CRDTs [#crdt]_, but these are
designed with different assumptions, and have different requirements on what
they need. To compare:

- Systems that use CRDTs do not generally transmit history, and apply operation
  and state-update events (for op-based and state-based CRDTs, respectively)
  outside of their intended context - i.e. the knowledge of the event's author,
  at the time that they generated that event. They are specifically designed
  around being able to ignore this context, i.e. to "move" these events around
  arbitrary places of recipients' histories.

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

  - In a state-based CRDT, the term refers to a partial order on the *states*.
    It (the order itself) has no "real representation" in terms of a data
    structure, but rather is a mathematical concept inherent to the definition
    of that particular CRDT, chosen independently of (and constant with respect
    to) any run of a protocol that uses it. Conceptually and in general, there
    *could* exist other mathematical partial orders over the same states, that
    also satisfy the same laws as required by the CRDT.

  - In the context of history-merge, the term refers to the partial order on
    the *events*. It (the order itself) is a data structure with real bits
    representing it, constructed during each run of the protocol. There can be
    no other partial order on the same events.

    It's *possible* (we haven't proved or checked it) that the laws we defined
    for 3-way-merge necessarily induce a partial order on the states. However,
    this is not directly relevant to any of our other discussions about our
    history-merge - and that is why we haven't bothered to prove or check it.

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
    TODO: link to Riak's set implementation which uses "casual context".
