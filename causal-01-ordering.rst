========
Ordering
========

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

Partial orders
==============

A `partial ordering`_ is a set of elements with a binary relation |le| that
is reflexive, antisymmetric, and transitive. It can be represented as a
`directed acyclic graph`_ (DAG), where nodes represent the elements, and
paths represent the relationships. Using this structure, we can begin to
model messages in a distributed system for a group communications session.

Each node m |in| M represents a message, sent by a sender u |in| U. [#Nsep]_
The relation a |le| b (a "*before*" b) means that the sender of b knew the
contents of a, when they wrote b. As a shortcut, let us also define |ge|
("after") such that a |ge| b |iff| b |le| a, and |perp| ("independent") such
that a |perp| b |iff| (¬ a |le| b |and| ¬ a |ge| b). Note that in a partial
order, ¬ (a |le| b) does *not* imply a |ge| b.

.. digraph:: relations

    rankdir=BT;
    node [height=0.5, width=0.5, style=filled, label=""];
    m [fillcolor="#ffffff", label="m"];
    p [fillcolor="#ff6666"];
    q [fillcolor="#ff6666"];
    s [fillcolor="#66ff66"];
    t [fillcolor="#66ff66"];
    x [fillcolor="#999999"];
    y [fillcolor="#999999"];
    z [fillcolor="#999999"];
    s -> m -> p;
    t -> m -> q;
    s -> x -> p;
    z -> q; t -> y;

In the diagram above, where an edge a |leftarrow| b means that a |le| b,

- red nodes (and m [#Neng]_) are *before* m; all others are *not-before* m.
- green nodes (and m) are *after* m; all others are *not-after* m.
- grey nodes are neither before nor after m, but *causally independent* of it.

Since |le| is transitive, a |le| b means that the sender of b knows all
messages before a as well. (We'll talk about how to enforce this in a real
system soon.) As a shortcut, define **anc(m)** = {a | a |le| m}, i.e. the
set of ancestors of m, including m. This set can be interpreted as the
possible "causes" of m.

In our scheme, each message m declares its immediate predecessors P =
**pre(m)**. This defines a few relationships: |forall| p |in| P: p < m.
(This is why we drew |leftarrow| for |le| in the above graph, rather than
|rightarrow| - the pointers belong to the later message; one cannot predict
in advance what messages will come *after* yours.) *Immediate* means that
there is nothing between them, i.e. |NotExists| q: p < q < m. One result of
this is that all parents (from now on we'll exclusively use this term) must
be causally independent of each other ("form an `anti-chain`_").

Our definition means that pre(m) forms a `transitive reduction`_ of |le|.
This eliminates redundant information from the contents of a message. This
is good, because redundant information is a security liability - it needs to
be checked for consistency, from the "canonical" sources of the information,
but then we might as well use those sources to directly calculate this
information. Here, enforcing immediacy and independence actually lets us
achieve some stronger guarantees "for free" - e.g., protection against
rewinding of vector clocks, a few paragraphs below.

In a real implementation, there are several options for representing nodes
and edges (i.e. messages and pointers to messages). One simple option, is to
broadcast the same authenticated-encrypted blob to everyone, treat this blob
as the content of the message, and use a hash function to reduce the content
into a pointer (identifier) that later messages can include. This ensures
that everyone uses the same identifier for a given message, and also that a
message cannot be changed later, since that would invalidate its identifier.
(Those familiar with git will see the similarities between this scheme and
git's representation of immutable commit objects.)

Possible extensions to this are discussed in the appendix. For example, we
could instead hash(blob||decrypted-verified plaintext), using some
definition for the plaintext that is the same for everyone. This ensures
that only people who can read the plaintext can create a valid pointer, but
it's not clear whether this is such a big deal. TODO: move to appendix and
discuss further.

.. _Partial ordering: https://en.wikipedia.org/wiki/Partially_ordered_set
.. _Transitive reduction: https://en.wikipedia.org/wiki/Transitive_reduction
.. _Directed acyclic graph: https://en.wikipedia.org/wiki/Directed_acyclic_graph
.. _Topological order: https://en.wikipedia.org/wiki/Topological_sort
.. _Anti-chain: https://en.wikipedia.org/wiki/Antichain

.. [#Nsep] Some other systems treat send vs deliver (of each message m) as two
    separate events, but we don't do this for simplicity. The sender
    delivers m to itself *immediately* after they send it, so the system will
    never see an event that is after send[m] but not-after deliver[m]; nor
    an event that is before deliver[m] but not-before send[m].

.. [#Neng] In natural language, |le| is really *before or equal to*. We say
    *before* here because that's shorter. We use |le| instead of < because
    that makes most formal descriptions shorter. Unless otherwise specified,
    we'll use *before* for |le| and *strictly before* for <, and likewise
    for *after* and |ge|.

Detecting replays, reorders, and causal drops
---------------------------------------------

To preserve causality, the system must deliver received messages (to higher
application layers) in `topological order`_. In other words, if we receive
m, but not yet delivered some p |in| pre(m), we must wait for p before
delivering m. Not doing this breaks the transitivity property of the parent
pointers, and results in more complexity (TODO: elaborate).

This already lets us detect messages that are received out-of-order, as well
as *causal drops* - drops of messages that caused (i.e. are *before*) a
message we *have* received. For the former case, recovery is straightforward
and we don't need to bother the user. For the latter, we can have a grace
period waiting for parent messages to arrive, after which we can emit a UI
warning ("timed out waiting for intermediate messages"), or perform recovery
techniques such as asking for a resend (covered in later sections).

If the first message is non-replayable (e.g. if it is fresh), we also
inductively gain replay protection for the entire session. This is because
each message contains unforgeable pointers to previous parent messages, so
everything is "anchored" to the first non-replayable message.

Detecting *non-causal drops* - drops of messages *not-before* a message
we've already received (and therefore we don't see any references to), is
more complex, and techniques for this will be covered in a later section.

Causal orders
=============

We can do a lot of things already with just a partial order, but adding some
extra structure results in stronger guarantees and enables more complex
behaviours. We do not make immediate use of this extra structure, but we
introduce them now, to identify some intuitive properties missing from a
partial order, and so that you are familiar with them later.

In a causal ordering, each message is associated with exactly one *sender*
and a set of *recipients*, collectively called its *members*. Define
**by(u)** as the set of all messages sent by u. This set is totally-ordered
on |le|, i.e. every element is either before or after every other. Roughly,
this constrains the number of branches that may exist at any point, and
helps the graph be "more linear".

Often, we'll find it useful to refer to the subjective session-view that a
sender has when they send a message. Define **context(m)** as a mapping
members(m) |rightarrow| anc(m), such that context(m)(u) is the latest
message by u seen by the sender of m, or |bot| ("null") if no such message
exists. Formally:

context(m): members(m) |rightarrow| anc(m) =
u |mapsto| max(by(u) |cap| anc(m))

where max() gives the maximum element of a totally-ordered set, or |bot| if
it is empty. We say a context c |sqsubset| c' ("strictly less advanced")
iff |forall| u: c(u) = |bot| |or| c(u) |le| c'(u) and c |ne| c'.

Context is structurally equivalent to a `vector clock`_. As with vector
clocks, malicious senders may "rewind" the context they are supposed to
declare with each message. If this redundant information is trusted, this
enables certain re-ordering attacks. TODO: give an example of this. To
protect against this, we define *freshness consistency*: all messages must
have a context that is strictly more advanced than the context of strictly
earlier messages. Or, in other words, a message may not declare a parent
that is *before* a parent of a strictly earlier message. Formally:

|forall| m', m |in| <: context(m') |sqsubset| context(m) (or equivalently)
|forall| p' |in| pre(m'), p |in| pre(m): ¬ p |le| p' (TODO: prove the equiv)

.. digraph:: freshness_consistency

    rankdir=RL;
    node [height=0.5, width=0.5];
    edge [weight=2];

    subgraph clusterA {
        label="by(A)";
        labeljust="r";
        2 -> 1;
        node [style=invis];
        r -> s -> 2 [style=invis];
    }

    subgraph clusterB {
        label="by(B)";
        labeljust="r";
        4 -> 3;
        node [style=invis];
        3 -> p -> q [style=invis];
    }

    edge [weight=1];
    3 -> 2;
    4 -> 1 [color="#ff0000", headport=se, tailport=nw];

Freshness consistency forbids the 1 |leftarrow| 4 pointer. This may seem like
quite a complex property to enforce, but actually we do not need to directly
enforce it.

**Theorem**: *transitive reduction* entails *freshness consistency*. Proof
sketch: if m' < m then |exists| p |in| pre(m): m' |le| p < m. Since
|le| is transitive, |forall| p' |in| pre(m'): p' |in| anc(p). By transitive
reduction, no other q |ne| p |in| pre(m) may belong to anc(p), and therefore
¬ q |le| p' (for all p', q) as required. []

So, we recommend that a real implementation should not encode context(m)
explicitly, since it is redundant information that can lead to attacks.
Instead, one should enforce that pre(m) is an anti-chain [#Nred]_, which
automatically achieves freshness consistency. Then, one may locally
calculate context(m) from pre(m), using the following recursive algorithm:
TODO: write this, perhaps in the appendix.

.. _Vector clock: https://en.wikipedia.org/wiki/Vector_clock

.. [#Nred] The merge algorithm, covered in a later section, includes a check
    that pre(m) forms an anti-chain.

Invariants
----------

Let us summarise the invariants on our data structure. Real implementations
may implement these as heavyweight unit tests.

- |le| is reflexive:

  |forall| a: a |le| a

- |le| is anti-symmetric:

  |forall| a, b: a |le| b |and| b |le| a |implies| a = b

- |le| is transitive:

  |forall| a, b, c: a |le| b |and| b |le| c |implies| a |le| c

- pre(m) is an anti-chain and forms a transitive reduction of |le|:

  |forall| m: |forall| p, p' |in| pre(m): p |perp| p'

  as above, this entails freshness consistency

- by(u) is a chain / total-order:

  |forall| u: |forall| m, m' |in| by(u): m |le| m' |or| m' |le| m

Linear ordering and display
===========================

Trade-offs
----------

Receive-first order
-------------------
