========
Ordering
========

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

Partial orders
==============

A `partial ordering`_ is a set of elements with a binary relation |le| that is
reflexive, antisymmetric, and transitive. It can be represented as a `directed
acyclic graph`_ (DAG), where nodes represent the elements, and paths represent
the relationships. Using this structure, we can begin to model messages in a
distributed system for a group communications session.

Each node m |in| M represents a message, sent by a sender u |in| U. [#Nsep]_
The relation a |le| b (a "before" b) means that the sender of b knew the
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

- red nodes (and m [#Nnat]_) are *before* m; all others are *not-before* m.
- green nodes (and m) are *after* m; all others are *not-after* m.
- grey nodes are neither before nor after m, but *causally independent* of it.

Since |le| is transitive, a |le| b means that the sender of b knows all
messages before a as well. (We'll talk about how to enforce this in a real
system soon.) As a shortcut, define **anc(m)** = {a | a |le| m}, i.e. the set
of ancestors of m, including m. This set can be interpreted as the possible
"causes" of m, like the discrete equivalent of a `light cone`_.

In our scheme, each message m declares its immediate predecessors P =
**pre(m)**. This defines a few relationships: |forall| p |in| P: p < m. (This
is why we drew |leftarrow| for |le| in the above graph, rather than
|rightarrow| - the references belong to the later message; one cannot predict
in advance what messages will come after yours.) *Immediate* means that there
is nothing between them, i.e. |NotExists| q: p < q < m. One result of this is
that all parents (from now on we'll exclusively use this term) must be causally
independent of each other ("form an `anti-chain`_").

Our definition means that pre(m) forms a `transitive reduction`_ of |le|. This
eliminates redundant information from the contents of a message. This is good,
because redundant information is a security liability - it needs to be checked
for consistency, from the "canonical" sources of the information, but then we
might as well use those sources to directly calculate this information. Here,
enforcing immediacy and independence actually lets us achieve some stronger
guarantees "for free" - e.g., protection against rewinding of vector clocks, a
few paragraphs below.

References must be globally consistent and immutable - i.e. to see a reference
allows one to verify the message contents, and no-one can forge a different
message for which the reference is valid, not even the original sender. One
implementation option, is to broadcast the same authenticated-encrypted blob to
everyone, treat this blob as the content of the message, and use a hash of the
message as the reference. (Those familiar with Git will see the similarities
with their model of immutable commit objects.) [#Next]_

There are two ways to know a valid reference - deriving it from the message
contents, or seeing it elsewhere (e.g. in pre(m) of a child). In the latter
case, one might not have seen the message itself. Then, one could pretend that
one *has* seen it, by re-using the reference in an outgoing message. We assume
that people won't do this - there is no strategic benefit to claiming that you
know something that you are entitled to know but temporarily don't, and the
rest of the system is designed to ensure that you see it eventually. The worse
that can happen is that someone will issue a challenge you cannot answer, but
only your reputation suffers in this case. Also, if you haven't seen the
message, you are unable to verify that the reference is indeed valid, so
someone else might be trying to trick *you*.

.. _Partial ordering: https://en.wikipedia.org/wiki/Partially_ordered_set
.. _Transitive reduction: https://en.wikipedia.org/wiki/Transitive_reduction
.. _Directed acyclic graph: https://en.wikipedia.org/wiki/Directed_acyclic_graph
.. _Topological order: https://en.wikipedia.org/wiki/Topological_sort
.. _Anti-chain: https://en.wikipedia.org/wiki/Antichain
.. _Light cone: https://en.wikipedia.org/wiki/Light_cone

.. [#Nsep] Some other systems treat send vs deliver (of each message m) as two
    separate events, but we don't do this for simplicity. The sender delivers m
    to itself *immediately* after they send it, so the system will never see an
    event that is after send[m] but not-after deliver[m]; nor an event that is
    before deliver[m] but not-before send[m].

.. [#Nnat] In natural language, |le| is really *before or equal to*. We say
    *before* here because that's shorter. We use |le| instead of < because that
    makes most formal descriptions shorter. Unless otherwise specified, we'll
    use *before* for |le| and *strictly before* for <, and likewise for *after*
    and |ge|.

.. [#Next] Possible extensions to this are discussed in the appendix. For
    example, we could instead hash(blob||decrypted-verified plaintext), using
    some definition for the plaintext that is the same for everyone. This
    ensures that only people who can read the plaintext can create a valid
    reference, but it's not clear whether this is such a big deal. TODO: move
    to appendix and discuss further.

Detecting replays, reorders, and causal drops
---------------------------------------------

Encoding the partial order already enables us to detect messages that are
received out-of-order. Actually, we completely ignore the receive-order, and
instead try to build up our data structure based on the parent references. When
we add a message to the structure, we emit a "delivery" event, which makes the
message available for subsequent operations, as well as consumption by higher
layers like the UI. (Sometimes we use the term "accept" when "deliver" might be
confused to mean "remote completion of a send", e.g. in "delivery receipt".)

For each message m, if any of its parents p |in| pre(m) have not yet been
delivered, we must place m in a buffer, and wait for all of pre(m) to be
delivered first. This enforces a `topological order`_ and acyclicity, and
ensures that if m was sent after p, then everyone else will see m after p. It
also allows us to verify that the parent references are actually of real
messages. (The sender should have already been authenticated by some other
cryptographic means, before we even reach this stage.) Since |le| is
transitive, in practise we deliver all of anc(m) before we deliver m.

The buffer should be an LRU cache with a maximum size, to prevent malicious
members from generating messages with non-existent parents that stay in the
buffer indefinitely. This is safe; even if valid messages are dropped, the
:doc:`consistency <02-consistency>` system will detect this and recover by
resending the dropped messages.

Withholding some messages even though they are available to deliver without
their ancestors, may seem like bad user experience. However, delivering them
like this sacrifices ordering and changes the original context of the message.
[#Nhld]_ As mentioned previously, our threat model is that this may be critical
for security; another interpretation is that messages without their ancestors
*are not* what the sender intended, so it doesn't even make any semantic sense
to deliver them. Our resend mechanisms should also reduce the cases in practise
where we must buffer messages.

One interface option is to show a "messages received out-of-order" notice. If
you really must display their contents, you must do this separately from the
main conversation interface, activated manually, and with a strong warning on
the consequences of reading them - they may be fake messages, do not reply to
them, etc etc. TODO: explore this further, with the requirements of SMS.

The structure also lets us detect causal drops - drops of messages that caused
(i.e. are *before*) a message we *have* received. We can have a grace period
waiting for parent messages to arrive, after which we can emit a UI warning
("timed out waiting for intermediate messages") or perform recovery techniques.
Detecting *non-causal drops* - drops of messages not-before a message we've
already received, and therefore we don't see any references to - is more
complex, and we will cover this in :doc:`freshness <03-freshness>`.

If the first message has replay protection (e.g. if it is fresh), we also
inductively gain this for the entire session. This is because each message
contains unforgeable references to previous parent messages, so everything is
"anchored" to the first non-replayable message.

.. [#Nhld] One major problem is we effectively break transitivity of the parent
    references. By which we mean, even if you somehow implement |le| despite
    having incomplete anc(m) and it passes transitivity tests, semantically
    this will not be true, so your representation will not reflect reality.
    Although these properties may seem abstract and not important to security,
    other more concrete properties that we elaborate on in further chapters,
    depend on them.

Causal orders
=============

We can do a lot of things already with just a partial order, but adding some
extra structure results in stronger guarantees and enables more complex
behaviours. We do not make immediate use of this extra structure, but we
introduce them now, to identify some intuitive properties missing from a
partial order, and so that you are familiar with them later.

In a causal ordering, each message is associated with exactly one *sender* and
a set of *recipients*, collectively called its *members*. Define **by(u)** as
the set of all messages sent by u. This set is totally-ordered on |le|, i.e.
every element is either before or after every other. Roughly, this constrains
the number of branches that may exist at any point, and helps the graph be
"more linear".

Often, we'll find it useful to refer to the subjective session-view that a
sender has when they send a message. Define **context(m)** as a mapping
members(m) |rightarrow| anc(m), such that context(m)(u) is the latest message
by u seen by the sender of m, or |bot| ("null") if no such message exists.
Formally:

context(m): members(m) |rightarrow| anc(m) = u |mapsto| max(by(u) |cap| anc(m))

where max() gives the maximum element of a totally-ordered set, or |bot| if it
is empty. We say a context c |sqsubset| c' ("strictly less advanced") iff
|forall| u: c(u) = |bot| |or| c(u) |le| c'(u) and c |ne| c'.

Unlike pre(m) and members(m), by(u) depends implicitly on the set of messages
that have been delivered so far, so might more accurately be denoted T.by(u),
where T is the current transcript. Though context(m) also refers to other
messages, any valid transcript must contain the image of context(m) due to the
topological delivery order, so it is unambiguous regardless of the transcript.

Context is semantically equivalent to a `vector clock`_. As with vector clocks,
malicious senders may "rewind" the context they are supposed to declare with
each message. If this redundant information is trusted, this enables certain
re-ordering attacks. For example, I should not be allowed to claim that my
last-received-from-A is 9, if I've already claimed that my last-received-from-B
is 12, but in message 12, B claimed that their last-received-from-A is 10. (A
more simple version of this with only 2 members is in the diagram below.)

To protect against this, we introduce *context consistency*: all messages must
have a context that is strictly more advanced than the context of strictly
earlier messages. Or, in other words, a message may not declare a parent that
is before a parent of a strictly earlier message. Formally:

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

Context consistency forbids the 1 |leftarrow| 4 reference. This may seem like
quite a complex property to enforce, but actually we do not need to directly
enforce it.

**Theorem**: *transitive reduction* entails *context consistency*. Proof
sketch: if m' < m then |exists| p |in| pre(m): m' |le| p < m. Since |le| is
transitive, |forall| p' |in| pre(m'): p' < p |equiv| anc(p') |subset| anc(p).
By transitive reduction, no other q |ne| p |in| pre(m) may belong to anc(p), so
q |notin| anc(p') |equiv| ¬ q |le| p' (for all p', q) as required. []

So, we recommend that a real implementation should not encode context(m)
explicitly, since it often has redundant information that can lead to attacks.
Instead, one should enforce that pre(m) is an anti-chain (see below), which
automatically achieves context consistency. Then, one may locally calculate
context(m) from pre(m), using the following recursive algorithm: TODO: write
this and link to the appendix.

.. _Vector clock: https://en.wikipedia.org/wiki/Vector_clock

Invariants
----------

Let us summarise the invariants on our data structure.

- |le| is reflexive:

  |forall| a: a |le| a

- |le| is anti-symmetric:

  |forall| a, b: a |le| b |and| b |le| a |implies| a = b

- |le| is transitive:

  |forall| a, b, c: a |le| b |and| b |le| c |implies| a |le| c

- pre(m) is an anti-chain and forms a transitive reduction of |le|:

  |forall| m: |forall| p, p' |in| pre(m): p |perp| p'

  as above, this entails context consistency

- by(u) is a chain / total-order:

  |forall| u: |forall| m, m' |in| by(u): m |le| m' |or| m' |le| m

The first three ought to be true independently of what data a message declares,
so may be verified within unit tests rather than as run-time checks. However,
the last two may be broken if we naively accept any message (which declares its
sender and direct parents) to be added to the data structure.

Enforcing transitive reduction can be done using only information from m and
anc(m). Since messages are only added to the data structure in topological
order, everyone has this information so they can enforce it themselves.
Messages that break this may simply be dropped, with a local UI warning as to
the source. The :ref:`merge algorithm for DAGs <merge-algorithm>` includes a
check that the inputs form an anti-chain, and it may be run on the pre(m) of
every message accepted, including for single-parent messages.

Enforcing total ordering can be done only when someone accepts messages from
both forks, which not everyone else might see. To help this propagate, users
that see this condition, must re-broadcast all messages within both forks to
everyone else, then leave the conversation with a BAIL error message (TODO:
elaborate possible errors) that references both forks. Note that one of the
references may have to be indirect if we already replied to the other fork, to
preserve transitive reduction. We should never need to do this for a triple
fork; bail should be immediate upon attempting to deliver a double fork. It is
not essential that everyone receives this, since lack of future participation
will indicate that something is wrong, but this helps to narrow down the cause.

Linear ordering and display
===========================

It's unclear the "best" way to draw a causal order - there are many layout
algorithms for general DAGs, that optimise for different aims. One approach we
believe is suited for group messaging applications, is to convert the causal
order into a total order (a linear sequence). These are simple to display, and
intuitive as a human interface for our scenario. To be clear, this artifical
order is only used for display; the core system, responsible for security
properties, works entirely and directly on the causal order.

Our first conclusion is that any linear order must not be the *only* interface
shown to a user. This is because *all* linear conversions are open to context
rebinding. The simplest case of a non-total causal order is G = (a |rightarrow|
o |leftarrow| b). WLOG suppose that our conversion turns G into (o, a, b) for
some user. Given only this information in a display, they cannot distinguish
(o, a) from (a, b) - yet a is not a real parent of b. This might lead to an
attack - if one user sees (o) and has a reasonable expectation that someone
will reply (b), then they may say (a) in the hope that (b) will be linearised
after it, rebinding its context.

To protect against this, implementions *must* distinguish messages whose pre(m)
does not equal the singleton set containing the preceding message in the total
order; and *should* provide a secondary interface that shows the underlying
causal order. This may be implemented as parent reference annotations, hover
behaviour that highlights the real parents, or some other UX innovation. For
example:

| Alice: innocent question?
| Chuck: incriminating question?
| Bob: innocent answer! [2]

The [2] means "the last message(s) the sender saw was the 2nd message above
this one". Messages without annotations are implicitly [1] - this has the nice
advantage of looking identical to the unadorned messages of a naive system that
does not guarantee causal order. In practise, only a minority of messages need
to be annotated as above; we believe this is acceptable for usability.

There are several caveats when selecting a linear conversion, and we need to
consider the trade-offs carefully. Let us introduce a few properties that would
be nice to achieve:

- globally-consistent / canonical: the conversion outputs the same total order
  for every user, given the same causal order input

- append-only: it does not need to insert newly-delivered messages (from the
  causally-ordered input) into the middle of a previous totally-ordered output.
  In a real application the user only sees the last few messages in the
  sequence, and earlier ones are "beyond the scroll bar", so messages inserted
  there may never be seen.

- responsive: it accepts all possible causal orders and may work on them
  immediately, without waiting for any other input

Unfortunately, we cannot achieve all three at once. Suppose we can. WLOG
suppose our conversion turns G (from above) into (o, a, b) for all users. If
one user receives (o |leftarrow| b), and does not receive (a) for a long time,
a responsive conversion must output (o, b). But then when they finally receive
(a), they will need to insert this into the middle of the previous output.

Delivery order
--------------

We believe that global consistency is the least important out of the three
properties above, and therefore sacrificing it is the best option. It's
questionable that it gains us anything - false-but-apparent parents are not
semantic, so it's not important if they are different across users. As noted
above, we ought to have a secondary interface to indicate the real causal order
*anyway*, to guard against context rebinding.

A linear conversion follows quite naturally from topics already discussed: the
order in which we deliver messages (for any given user) is a total order, being
a topological sort of the causal order. It is practically free to implement -
when the engine delivers a new message, we immediately append it to the current
display, and record the index for later lookup. This conversion is responsive
and append-only, but not globally consistent. It is also nearly identical to
existing chat behaviours, save for the buffering of dangling messages. [#Nres]_
Given its simplicity and relatively good properties, we recommend this as the
default for implementors.

Another approach achieves global consistency, but sacrifices one of the other
properties: bounce all messages via a central server, which dictates the
ordering for everyone. This strategy is popular with many existing non-secure
messaging systems. If we embed the causal order, we can re-gain end-to-end
security, and protect against the server from violating causality or context.
Then, we must either sacrifice responsiveness by waiting until the server has
reflected our message back to us before displaying it, or sacrifice append-only
and be prepared to insert others' messages before messages that we sent and
displayed immediately.

In either case, we still need *another mechanism* to ensure that the server is
providing the same total ordering to everyone, the whole point of doing this in
the first place. TODO: outline a mechanism for this. Or, we could omit this
mechanism and fall back to "delivery order" as above; then "global consistency"
would be achieved on a best-effort basis, relying on the server being friendly.

.. [#Nres] One may argue that due to this buffering, we are sacrificing
    responsiveness; however there is no *additional* waiting beyond what is
    necessary in order to achieve causal consistency - as discussed before,
    sacrificing the latter results in less security and greater complexity.
