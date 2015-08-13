===========
Consistency
===========

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

One necessary property of a group session is that everyone sees the same
transcript - that is, the same set of messages and their relative orders.

Everything we mention here will work under both synchronous and asynchronous
transports. Though the specific values of certain time-based variables and
distributions change depending on this, the strategies that we propose will
adapt to any value, i.e. work regardless of the values.

.. _reliability:

Reliability and consistency
===========================

Consistency can be viewed as a case of metadata verification. Each message has
an author, a context, and some recipients. We verify the author with message
authentication, verify context with parent hash references and by checking
their transitivity and ordering properties, and now we need a way to verify
the (other) recipients, a.k.a. ensure consistency.

Consistency requires reliability. If some messages aren't actually received by
everyone, then of course we can't possibly verify that they were. Or from
another angle: if we don't provide reliability, then consistency checks will
emit so many false positive warnings due to unreliability, that the former
becomes useless to the user as a security indicator.

Earlier proposals for consistency in group messaging protocols, have typically
involved running a process at the end of a session, to ensure consistency for
the whole session. This process is simple to reason about, but not explicitly
robust against any lack of reliability, and does not provide useful information
during the middle of a session.

We prefer an incremental model of consistency, where we continually verify the
other recipients (i.e. consistency) of messages as we accept them, and also
optionally provide reliability if we need to. Below, we describe our strategy
for achieving this, using the causal order from the previous chapter, that has
low communication overhead comparable to said earlier proposals.

We first observe that a message by itself (including metadata), fundamentally
does not contain any information on whether its claimed recipients *actually*
received it - since any author may author a valid message and simply not send
them to some of the claimed recipients, perhaps with the co-operation of the
transport. Therefore, verifying these recipients must be done strictly *after*
we accept the message, i.e. *after* we verify the author via MAC or signature.

Further, if our transport breaks and we receive no more messages, consistency
must not default to "OK". That is, our mechanism must start at a "MAYBE-BAD"
state, then warn the user if it does not turn into an "OK" state after some
time. So we need a timer primitive; we cannot *only* run consistency checks as
part of a packet receive-handler.

Before we jump straight into consistency, we'll look at reliability first.
Brief overview of TCP; "ack seqnum", resends of higher seqnum packets. Our
strategy is similar. TODO: Why not simply use transport reliability? See
:ref:`transport-reliability` for why not.

.. _full-ack:

Acks and full-ack
=================

In this section, we talk about our **data structures** and **concepts**. We
interpret parent references as an implicit acknowledgements of the parent, and
explore what this means for reliability and consistency.

When we see a message, we want to be sure that everyone else also saw it.
Rather than trying to enumerate and detect all possible failures, we aim for a
certainly-good state. Conveniently, we already have the mechanics for this in
our causal order, namely the property that every message refers transitively to
all its ancestors, via parent references.

We can be sure that r saw m, if we see a message am by r that refers to m,
perhaps indirectly. We call this an acknowledgement, "ack" for short. This can
be *any* message that refers to m, not necessarily a special ACK message; the
references are what's important here. Now, we have two cases to consider:

- We sent m:

  - We assume that we ourselves sent the message consistently to everyone.
  - We expect that all the recipients will ack the message, so that we know it
    was reliabily sent.

- Someone else sent m to us (and some others):

  - We must ack the message ourselves, so that:

    - the author knows it was reliably sent
    - the other recipients know it was consistently sent.

  - We expect that the other recipients will ack the message, so that
    we know it was consistently sent.

In both cases, once a message is acked by all its recipients, and we (i.e. from
the view of a particular member) see these acks, we are sure that it has been
sent both consistently and reliably. We call this state *fully-acked*, and
afterwards we no longer need to worry about the message. Note that we don't yet
know that others have also reached "full-ack", which is a :ref:`harder problem
<consistency-vs-consensus>`. But this is OK, as argued in more depth later - we
never actually need to know this.

In more precise terms: given a valid transcript T, we define that a message m
is **acked-by** r iff |exists| m' |in| by(r): m < m', and is **fully-acked**
iff |forall| r |in| recipients(m): m is acked-by r. [#Npvis]_ Note that m
becomes fully-acked at the same position for everyone [#Nfack]_, namely just
after T accepts all of its direct acks **diracks(m)** = { min(by(r) |cap|
des(m)) | r |in| recipients(m) }, des(m) being the descendant messages of m,
and min(|emptyset|) = |bot|. Notice the similarity to context(m) - one may
think of these as a "future" context(m). If and when full-ack is reached, this
will contain no |bot| values. [#Nimpl]_ To visualise this:

.. digraph:: acks

    rankdir=RL;
    node [height=0.5, width=0.5, style=filled, label=""];
    m [fillcolor="#ff9999"];
    p [fillcolor="#99ff99", color="#ff0000", label="p"];
    q [fillcolor="#9999ff"];
    r [fillcolor="#ff9999", color="#ff0000", label="r"];
    s [fillcolor="#99ff99", color="#ff0000", label="s"];
    r -> p -> m;
    r -> q -> m;
    s -> q;

Background colours indicate authorship, red borders indicate messages that are
not yet fully-acked. If blue then sends a message referring to r, s, it will
cause p to become fully-acked, and help r, s along on the way towards being so.
This is an incremental consistency; full-acks occur as new messages arrive,
instead of all at once at the end of the session.

.. [#Npvis] This definition becomes slightly more complex when we introduce
    :doc:`partial visibility <05-visibility>`; see that chapter for details.

.. [#Nfack] That is, position in the graph, not necessarily the real clock time,
    which we don't deal with until later. One way to visualise this is as a
    "cut" in the graph just after diracks(m), that filters out later messages.

.. [#Nimpl] For space-efficiency, one should track *unacked* recipients instead
    of acked ones: when accepted, each message has a set unackby(m) that is
    equal to recipients(m), and it gradually becomes empty as later messages
    (that ack it) are accepted. When the set becomes empty, its ack-monitor is
    cancelled and a "fully-acked" event is emitted.

Ack monitors
------------

As mentioned before, we expect all messages to become fully-acked, so we need
an active process outside of the send/recv logic to manage this and warn or try
to recover if it takes too long. We call this an "ack-monitor"; we activate a
new one for each message accepted, and de-activate it on full-ack. The basic
behaviour is to warn the user if full-ack is not reached within a grace period
and cancel/reduce this warning if it is reached afterwards, with some nuances
and exceptions that we're about to go into. Later on, we'll use it to implement
a primitive reliability mechanism as well.

For everyone to reach full-ack, each member must send an ack, for each message
they received and did not send themself. But it is very wasteful for everyone
to send an empty message simply to ack previous messages that have content;
this would multiply the number of packets sent. So instead, we'll come up with
a more efficient strategy.

Define an **explicit** ack as a message whose only purpose is to acknowledge
previous messages via its parent references, with no content or other semantics
in the body of the message, and an **implicit** ack as a message with a body,
that incidentally acknowledges previous ones. Explicit acks should not be
displayed as a message in the UI, but it is still kept in the transcript graph,
as a message object that differs from other message objects in its type and the
fact that it has no body.

We observe that, in a typical session, a member will probably send a message
after a certain period *anyway*, which is an implicit ack of previous messages.
Then, we (as their local process) don't need to send an explicit ack on behalf
of them. So, in our system, we may set our full-ack warning timeout roughly to
this expected time-to-reply, and just wait this out. Then, during a lull in the
session when there are no replies after the timeout, or at its end when members
commit to not sending any more messages, we may fall back to sending explicit
acks automatically. This most naturally done within the ack-monitor, since that
runs the timeouts.

Since explicit acks carry no other semantics, we don't care if they are not
fully-acked, so we don't need to activate a new ack-monitor for it. That is, we
don't need to warn the user if it is not fully-acked after a timeout, nor to
send a further automatic explicit ack to ack it. [#Nainf]_

.. [#Nainf] If we did auto-explicit-ack an explicit ack, this would result in
    an indefinite sequence of mutual acks. This can help to achieve freshness
    but is not directly useful for consistency so we'll skip it for now; in the
    next section we deal with freshness in more precise and quantifiable terms.

Ack semantics
-------------

TODO: reword this section

An implicit ack, such as a normal user message, indicates "some" level of
understanding of previous messages. [#Nread]_ Automatic explict acks *should
not* be interpreted to carry this same weight, because the user has no control
over whether they actually read those messages or not. If one desires an
explicit "user-level" ack (e.g. in critical situations) there are a few
options:

- a manual explicit ack, that must be initiated by the user - like acks in
  `Pond`_, which are just empty messages.

- a pseudo-manual explicit ack, that may be interpreted like a manual explicit
  ack. This is triggered automatically, but only when the user is interacting
  with the application, or has it focused in the foreground.

These would be implemented as a supplement to the automatic ack. Other
projects' terminology for these concepts include "delivery receipt" for
"automatic ack" and "read receipt" for "manual/pseudo-manual ack".

.. _Pond: https://pond.imperialviolet.org/tech.html

.. [#Nread] The user could avoid reading the messages, but we can't do anything
    about this, and it's dubious that this could be abused for benefit. In the
    *average* case, there is some understanding.

Timing concepts
===============

We generally avoid require consideration of time periods, and prefer to reason
about the relative ordering of events. However, as described previously, some
parts of our design could not be achieved without this, namely grace periods
and warning timeouts. Let's attempt to define these more precisely now. There
is a lot of depth here, and we won't go all the way down the rabbit hole, but
it is useful to sketch out a semi-formal framework for reasoning about these
concepts, so that we can build an initial concrete software implementation that
can be extended later without re-writing everything.

Here are some concepts that form a foundation for our later ideas:

``BC_LAT``:
    Broadcast latency. This is the time taken for one packet to reach all of
    its recipients. In a 1-to-1 communication context, this is equivalent to
    half of the round trip time (RTT).

``M_INTV``:
    Message interval. This is the time in between successive messages sent by
    a single member. In a real session, this would probably increase as the
    number of members increases.

``FA_INTV``:
    Full-ack interval. This may be defined as ``2 * BC_LAT + M_INTV``, assuming
    that each member will, if they don't implicitly ack a message, send an
    automatic explicit ack of it after ``M_INTV``.

The base concepts are random variables, that take different concrete values for
particular instances of a relevant event. For the purposes of our messaging
system, we are interested in picking a cut-off value s (which may depend on the
environment and not necessarily be constant) below which we deem an event to be
"innocent", and above which we'll treat as abnormal and maybe serious enough to
emit a user warning, such as "not fully-acked" for ``FA_INTV``. When we refer
to an ``ALL_CAPS`` name, it will hopefully be clear from the context whether we
mean a single event, the distribution of events, or the cut-off value.

For each cut-off value s, a basic simple estimate would be to model the
distribution P(t = s), where t is the r.v. on the time it takes for the event
to happen, and pick s to be the 95% percentile value for t.

A more precise treatment would model the joint distribution P(R, T(s)), where R
is a r.v. on whether reality is good (we are not under attack) and T(s) is a
r.v. on whether our test passed (t < s) or failed (t > s), and choose s to
maximise some information retrieval metric (e.g. precision, recall, f1-score)
defined in terms of the distribution. However, we would need lots of data and
complex ML inference techniques to do this properly, so we skip it for now.

Resends and duplicates
======================

In this section, we talk about our **algorithms** and **execution strategy**.
We look at a fundamental building block of reliability mechanisms, that is to
actively resend messages, as well as how to react to duplicate messages.

Our basic reliability mechanism is to periodically resend the packet for a
message m, until m is fully-acked. It seems most natural to implement this
behaviour in the ack-monitor for m, since this detects the full-ack condition.

However, explicit acks do not have ack-monitors on them, so we must guarantee
their reliability a different way. We'll do this by having each ackmon resend
not only its subject message m, but also chains of explicit acks sent directly
before it, namely { p | p |in| anc(m) |and| EA(p) |and| |NotExists| q: p < q <
m |and| ¬ EA(q)} where EA(x) is true if x is an explicit ack.

This still misses out explicit acks that have no implicit acks after them; we
cover this last case indirectly by how we deal with duplicates.

Define a **duplicate** packet as one that verifies-decrypts to a message with
the same metadata (author, parents, recipients) and data as an already accepted
message. [#Ndupq]_ Suppose we receive a duplicate packet d from sender s for a
message m. This is because s hasn't yet reached full-ack on m:

- If we authored m, there is nothing more we can do to help s reach full-ack.
- Else, someone else authored m, and we are:

  - not meant to ack it (e.g. it's an explict ack), so have nothing to do.
  - meant to ack it, and have set an ackmon on it.

    - If we haven't yet acked it, the ackmon will ensure that we ack it.
    - Else, we have acked it with message am, but s has not received this.
      Then, they won't ack am either. Either am is:

      - meant to be acked, and we have set an ackmon on it. Then, it will
        detect that s hasn't acked am, and resend it automatically. (Or s
        already acked am, and d was received due to a transport glitch or an
        attempted replay attack. In any case, we take no action here.)
      - not meant to be acked, so we didn't set an ackmon on it. But we have no
        other process to resend it, so we should resend it now.

The reliability scheme outlined above is quite implicit - we decide when to
resend packets using inference based on unmet expectations, and avoid explicit
"request resend" messages. More research is needed to optimise performance; in
practise, we are using an exponential backoff algorithm with base interval
``BC_LAT``, but first wait for ``FA_INTV`` before starting the iterations, with
an extra throttle to reduce multiple members from resending the same packet
many times in quick succession. This has worked adequately, and our underlying
scheme has a clear logical justification, so we'll continue with other topics.

.. [#Ndupq] Duplicates of non-accepted queued messages may be ignored - we
    don't even know if it's a valid message yet, and if/when we do, the rules
    for duplicate accepted messages will handle that.

.. _single-ciphertext:

Single-ciphertext principle
---------------------------

To simplify resends, we suggest to follow the "single-ciphertext principle" -
each message is communicated as the same ciphertext for everyone, and this
applies even if parts of it are encrypted to only a subset of the recipients.
This makes it easy to detect duplicate resent messages, and also allows a
member to resend a message authored by someone else, which is useful if the
latter is absent. All members cache this ciphertext in case they have to resend
it to others. When the message is fully-acked, the cached ciphertext may be
deleted to save space. This should not affect security (e.g. forward secrecy)
of the data, since the threat model already includes an eavesdropper that
stores all ciphertext.

TODO:
- refine auto-deletion of old ciphertext that has been full-acked (cf code)

If an entity - e.g. an XMPP server - can observe multiple recipients receiving
the same ciphertext, then this links these recipients. Hiding this metadata is
outside of the scope of this chapter, and adhering to the single-ciphertext
principle makes this leak no worse than existing systems, but newer systems
that try to explicitly provide unlinkability will need to think about this.

Transcript consistency
======================

Straightforward after "full-ack". Similar to TCP - send FIN.

Future consistency, consistency-on-leave.

Transcript consistency may be constructed simply out of our message consistency
primitives. When we want to part, we send a "intend-to-part" (FIN) message zm
then wait for explicit acks to it. Once it is fully-acked, we reach consistency
for all of anc(zm), which we'll treat as the transcript. Recipients should ack
these immediately, instead of the usual behaviour of waiting a grace period for
the user to manually send an implicit ack. Since there won't be any future
chance to receive the acks, the sender may wait longer than usual for full-ack,
some small multiple of ``BC_LAT``.

If we receive implicit or explicit acks to any message other than zm, we must
ignore them or display/log them permanently with a warning, because we have no
chance to verify their consistency. The same goes for implicit acks to zm.

If we fail to reach full-ack for zm (i.e. time out waiting for it), then we
still have consistency for all the messages we had previously reached full-ack
for, which hopefully is almost equal to anc(zm). For all other messages, we
should display/log them permanently with a warning.

TODO: construct a similar mechanism for cases where someone else wants to force
us to part the session.

This ensures consistency for messages that we delivered locally, including ones
we sent. What about messages by others that we haven't delivered, including
ones we haven't even received? We'll talk about this in the next section.

TODO: mention second-order knowledge and link to below section

Future extensions
=================

Use ML to tweak flow control, just like we have various complex TCP flow
control algorithms. Probably not necessary for "text messaging" low-bandwidth
applications.

Selective acks as in TCP. ("extra-mIds seen" on top of parents?)

.. _transport-reliability:

Rejected ideas
==============

Transport-level reliability
---------------------------

TODO: reword

Motivation, can't use transport-level reliability.

- not authenticated

- not relevant *to the application* (c.f. "internet smart at the edges", but
  the edges means internet edges, not application edges. e.g. third party
  MUC chat servers are "smart at the edges" of the internet, but we don't care
  about this for application security purposes.)

- *cannot* be relevant to the application - for efficiency, performance,
  resource limitation, same reason why internet routers don't try to provide
  reliability to a TCP connection.

- unnecessary failures, false positives, liveness vs security

- ack failures necessarily happen *after* delivery, but typical reliability
  mechanisms don't communicate this to higher layers

It is quite common for cryptographic systems to ignore reliability, and assume
it will be covered by a lower layer. The universal prevalence of TCP today
probably encourages this. However, the group messaging scenario introduces some
failure modes that cannot be recovered from by a lower layer, and this section
will explain these and propose a fix. Such failures will eventually cause a
failure of our consistency checks, so it is "safe" to ignore reliability; but
unnecessary failures are bad user experience and may incentivise people to
choose a less secure application, so should be avoided where possible.

To demonstrate what we mean by "unnecessary failures", we'll talk briefly about
existing reliability mechanisms. Typically, these will continually resend a
packet until it gets a transport-level ack from the recipient. These are not
authenticated cryptographically, so transport-level delivery claims are not
to be trusted, which is why we have message-level acks above. A malicious
transport-level attacker can also just drop any packets to break reliability.
However, there are other failure modes that *honest* transports can't recover
from, but that *can* be recovered from at the end-to-end message level.

TCP doesn't communicate ack failures to higher layers. If your partner drops
their connection right after your message is passed to TCP, an ack will never
be received. Our application might automatically start a new TCP session later,
but this won't know about the lost message from the previous session, resulting
eventually in consistency failure. In the instant messaging case, these types
of failure may be rare, but they are still unnecessary. And they become much
more common and problematic if we want to support asynchronous messaging, where
not all users are online at once - there is no Internet standard for a generic
reliable asynchronous transport. [#Neml]_

With transports that run to a third party, such as a TCP session to a central
reflector server, the transport layer does not detect end-to-end reliability
failures, even if the server is honest. No matter how hard the server tries to
guarantee simultaneous presence, some clients may be offline when a packet is
first sent. Some servers try to mitigate this, by resending packets to those
offline clients when they come back online. In XMPP, several mechanisms exist
that approximate this: `session resumption`_ for short transport failures, and
`discussion history`_ as ad-hoc opportunistic context on joining a channel.
However, in both cases the scope of reliability is limited, either time-wise or
packet-count-wise. This is an inherent limitation of public servers: they are
unable to offer *unconditional* end-to-end reliability since this requires
unconditional buffering of unacked packets, but that may be abused by malicious
clients resulting in denial of service. By contrast, clients have the incentive
to indefinitely buffer their own not-fully-acked messages.

That is not to say transport-level, or third-party, or non-authenticated,
reliability schemes are useless. They are quite cheap to run, and recover from
most failures. But the non-recoverable failure rate is much higher in certain
environments or for certain use-cases, so it is prudent to implement a more
expensive but more robust authenticated end-to-end scheme.

.. _Session resumption: http://xmpp.org/extensions/xep-0198.html#resumption
.. _Discussion history: http://xmpp.org/extensions/xep-0045.html#enter-history

.. [#Neml] Email resending is application specific, has a lot of baggage, and
    is generally considered unsuitable to build clean applications on top of.

.. _consistency-vs-consensus:

Consistency vs consensus
------------------------

TODO: reword

In our usage, consistency is *not* consensus. Consensus is about agreeing on a
future value or action, that has not been decided yet; consistency is about
agreeing on past history, that has already happened. So we don't need to use
Byzantine fault tolerance techniques (which have probabilistic success), and
can achieve a stronger guarantee with simpler techniques.

With consistency, we only achieve "first-order" knowledge - that is, for each
message m, from the view of a member u, u knows (P) everyone has seen m, but u
doesn't know if (Q) others know P, nor if (R) they know u knows P, etc. That's
OK; we don't foresee any situation where we need those latter properties, so
they are lesser concerns. For comparison, consensus requires us to achieve
`common knowledge`_ (i.e. "infinite-order" knowledge).

We believe that this is not a security problem - it is the responsibility of
each user to check that all messages they see are fully-acked. If our ack is
dropped, then even though we don't know they have received our ack, we know
that they will not treat the un-acked (from their view) message as part of the
consistent transcript - it is not in their interest to do so. So we don't need
to *guarantee* that our own explicit acks are fully-acked. This avoids the
Byzantine agreement problem.

A related concept that *does* provide a guarantee for explicit acks, is
:ref:`heartbeats <heartbeats>`. These can be thought of as automatic explicit
acks that result in an infinite sequence of mutual acks. But their purpose is
to ensure freshness rather than consistency, and their cost may be unsuitable
for some asynchronous protocols. So we omit them here, and talk about them
separately as an optional feature.

.. _Common knowledge: https://en.wikipedia.org/wiki/Common_knowledge_(logic)
