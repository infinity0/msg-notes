========
Ordering
========

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

Partial and causal orders
=========================

A `partial ordering`_ is a set of elements with a binary relation |le| that
is reflexive, antisymmetric, and transitive. It can be represented as a
`directed acyclic graph`_ (DAG), where nodes represent the elements, and paths
represent the relationships. [#Nred]_

Using this structure, we can begin to model messages in a distributed system
for a group communications session.

Each node m |in| M represents a message, sent by a sender u |in| U. [#Nsep]_
The relation a |le| b (a "*before*" b) means that the sender of b knew the
contents of a, when they wrote b. As a shortcut, let us also define |ge|
("after") such that a |ge| b |iff| b |le| a. Note that in a partial order, ¬
(a |le| b) does *not* imply a |ge| b. That is, we can talk about *before*
and *not-before* separately from *after* and *not-after*.

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
**pre(m)**. This defines a few relationships: |ForAll| p |in| P: p |le| m.
*Immediate* means also that there is nothing between them, i.e. |NotExists|
q: p |le| q |le| m. (This is why we drew |leftarrow| for |le| in the above
graph, rather than |rightarrow| - the pointers belong to the later message;
one cannot predict in advance what messages will come *after* yours).

We choose only to declare immediate predecessors, in order to eliminate
redundant information from the contents of a message. TODO: expand why this
is good from a security POV.

In a real implementation, there are several options for representing nodes
and edges (messages and pointers to messages). One simple option, is to
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
.. _Covering relation: https://en.wikipedia.org/wiki/Covering_relation

.. [#Nred] Technically, it is the `transitive reduction`_ that is represented
    by the graph, where each edge represents a `covering relation`_.

.. [#Nsep] Some other systems treat send vs deliver (of each message m) as two
    separate events, but we don't do this for simplicity. The sender
    delivers m to itself *immediately* after he sends it, so the system will
    never see an event that is after send[m] but not-after deliver[m]; nor
    an event that is before deliver[m] but not-before send[m].

.. [#Neng] In natural language, |le| is really *before or equal to*. We say
    *before* here because that's shorter. We use |le| instead of < because
    that makes most formal descriptions shorter. Unless otherwise specified,
    we'll use *before* for |le| and *strictly-before* for <, and likewise
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

Invariants
----------

Before we move further, let us define some additional constraints on top of
our partial order. Recall that in a partial order, |le| is reflexive,
anti-symmetric, and transitive. In a causal order, where each message is
associated with a sender u, we have some additional structure:

- send total-ordering: all messages from each sender are totally-ordered.

  Formally, for all u: for all m, m' in msg(u): m |le| m' or m' |le| m

- freshness-consistency: a message m cannot declare a parent that is
  strictly before a parent of a previous message m' < m.

  Formally, for all m, m' in G: m |le| m' |Implies| anc(pre(m)) |sube|
  anc(p') for all p' in pre(m'). TODO: this is wrong, correct it

Another way of saying this is that the context always advances. TODO: define *context* here.

We can do a lot of things already with just the partial-order structure, but
the causal-order structure offers better guarantees and enables more complex
behaviours, so we thought we'd introduce it early.

Linear ordering and display
===========================

Trade-offs
----------

Receive-first order
-------------------
