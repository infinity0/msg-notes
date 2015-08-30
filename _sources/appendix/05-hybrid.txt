===============
Hybrid ordering
===============

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

Causal orders have some nice properties explained elsewhere, but are also hard
to implement correctly when not everyone is allowed to see the full history.
(At the time of writing, we don't have a full solution for this yet.)

One alternative is to have a total order for membership operations, while
leaving messages to be in a causal order. Here, we present a scheme for this
that requires no extra client packets, but does require one extra round of
latency - the minimum necessary for a total order that preserves context.

The scheme only applies to certain membership operations packets - for example,
messages in the causal order of a session transcript may instead be sent over a
different transport that offers more protection against metadata analysis. Such
hybrid strategies may be researched in further depth elsewhere.

Models and assumptions
======================

Before we begin to describe our scheme, we'll clarify our assumptions about the
context that we're working in. The models we define are not completely formal,
but should be precise enough to understand the rest of the chapter with.

Group transport channel
-----------------------

We model the transport channel as a globally consistent sequence of events,
where each event at index i is associated with a set of *channel members*,
denoted by mem[i]. An event may be a packet [#Npket]_ with mem[i] = mem[i-1],
or else it is a member-change, that explicitly encodes a change to the channel
membership. A packet is visible to mem[i], and a member-change is visible to
both mem[i-1] and mem[i]. (The indexes are only indicative, for the purposes of
this document, and need not be visible to the members nor otherwise exposed.)

From each member m's point of view, they see a member-change that adds them to
the channel, some packets or member-changes *not* acting on themself, then a
member-change that removes them from the channel. The next event, if any, would
be a member-change that adds them to the channel again, and this cycle can
repeat any number of times.

These events need not be authenticated, at this layer, between the members.
Further, this model only constrains events *received* by members, and is
neutral to how events *sent* by members are handled. For example, a member may
send a packet P immediately after he receives event i with mem[i] = M, i.e.
intending to send P to M, but P may be added to the channel event sequence at a
later index j > i+1 where mem[j] |ne| M. Our scheme may be viewed as a
user-level (end-to-end) method of addressing these security deficiencies of the
transport, which cannot be satisfied on the transport layer itself.

It is not necessary to specify exactly the mechanism for achieving this model.
However, a typical scenario would be a dumb server that accepts all events from
all clients, and echoes them back reliably to everyone in the same total order.
This is quite standard for existing instant messaging protocols such as XMPP,
and requires no extra logic on the server side. Therefore, clients can
transparently run our scheme on top of existing infrastructure. [#Nprot]_

What we *do* expect however, is that the mechanism takes responsibility for
delivering, reliably and in a consistent order, events to its relevant visible
users. We do not *assume* this to be the case; any deviation from it is
detected by our scheme, and regarded as incorrect behaviour of the mechanism
(e.g. the server). But in the case where they are met, the scheme works under
these assumptions to be efficient and robust.

For our hybrid order scheme, we only require that certain packets are sent over
this channel, i.e. initial packets and final-candidate packets, defined later.

.. [#Npket] We use the term "packet" to refer to transport-level packets; the
    term "message" refers specifically to verified-decrypted messages, that
    comprise the causal order history graph.

.. [#Nprot] Note that some protocols may use different concrete sequences of
    protocol "event" packets, but logically these are equivalent to the above
    model. For example, in XMPP an "I entered the channel" event, in practise
    consists of many enter events for each member in the channel, from just
    before I enter it, then a single enter-event of me entering the channel.
    One can write software adapters to translate between these differences.

Group membership operation
--------------------------

We model the operation as a process G on the (logical) session membership. G is
initialised from some previous state - either a null state, or the result of
the previous operation - that encodes the previous session membership, M'; and
some initial packet pI received from the channel, sent by a member of M', which
defines the intended next session membership, M. [#Noini]_

Depending on the packets received by G, the operation may finish with success
("completes"), upon which the session membership atomically changes to M and
the result state is stored for the next operation; or with failure ("fails"),
upon which it remains at M' and all temporary state related to G is destroyed.

We assume that G handles authentication and ordering of its own packets, that
only fresh packets (relative to pI) are accepted, and that packets will not be
misinterpreted to be part of another operation, that has a different initial
packet. If the operation finishes, then this is authentic; if it completes,
then any resulting state is also authentic. G defines the subject of these
authentications - e.g. by all members in M, or only by the initiator. We
assume that the people who choose which particular G to plug into our scheme,
make a sensible decision that fits their security requirements.

A corollary is that packets that belong to different operations have different
content (i.e. are unique). This should not be difficult; indeed otherwise it
would be susceptible to replay attacks. Duplicate packets should be ignored.

For a general G, the packets required to finish the operation may be ordered
differently across members. For example, in a final round where everyone must
respond to previous values, it doesn't matter what order we receive these in;
G can still complete in the same way for everyone.

However, for our scheme to remain relatively simple, we place an additional
restriction. Define a "final-candidate" packet as one which *may* finish G in
some linearisation of the causal order of G's packets. Then, we require that
these packets must be sent over the group channel, as opposed to out-of-band.
We use this fact to define an implicit agreement mechanism, so that members can
determine which particular packet finishes G. We call this final packet pF.
Given a channel event sequence that "almost" finishes G, there may be several
possible packets appended to this, that could each actually finish G; we call
these "final packet proposals"; the agreement mechanism defines which one is
actually accepted, and hence the result of the operation.

As with everything in the channel, pF is defined only after the packet is
*received*. For example, suppose we have a G that expects a final round of 3
packets {F[A], F[B], F[C]}, and C receives F[A] and F[B]. Now, she has all
information needed to generate F[C] and complete the operation, but she must
not execute the completion yet, since there may be an fail packet A added to
the channel event sequence *before* F[C]. In this case, everyone including C
must fail G, and pF is A and not F[C].

In terms of who is able to decode and identify initial or final packets, we
assume that:

- | for all pI with a given prev_pF: only { all of M'\|M } can identify it, or
  | for all pI with a given prev_pF: everyone can identify it
- | for all pF with a given prev_pI:
  | if pF success: only { all of M'\|M } or only { all of M } can identify it
  | if pF failure: only { all of M'\|M } can identify it

No other partitions are allowed, i.e. a process G may not be such that e.g. for
a pIa with a given prev_pF, M'\|M can identify it, but for a pIb on the same
prev_pF, everyone can identify it; nor that e.g. some members of X := M'-M can
identify a pF, but others cannot.

For the pI case, the arrangement of the { or, for all } quantifiers simplify
the cases we have to analyse. For the pF case, the arrangement is more complex,
but the simpler form would be *too* restrictive, as explained later. Typical
real-world instances of G already fit these constraints; we just specify them
more precisely here to benefit further discussions. These constraints are
justified further in :ref:`hybrid-partial-vis`.

.. [#Noini] If the membership protocol supports outsider initiation (i.e. join
    as well as invite), it is the *reply* to the join request that is treated
    as the pI within our scheme here. This fits our requirement that pI be
    authored by member of the previous session (if any), who knows the previous
    state associated with it.

Ordering and consistency
========================

.. _hybrid-context-preserve:

Context preservation
--------------------

By our assumptions of G, the context of a final packet is already preserved,
since G will only interpret it within the scope of the initial packet. To
preserve the context of an initial packet, we add to it explicit references to
that context, i.e. the following information:

prev_pF:
  packet-id (defined later) of last accepted pF
pmId:
  latest message-ids seen, in sub-session derived from pF

This lets us convey our context to R := M'&M, the members that remain in the
session. It is unnecessary to convey it to excluded members, since they are no
longer part of the session; nor included members, since the context is not
visible to them. [#Npctx]_ If there was no previous sub-session, then we have
neither context, nor anyone to convey it to (since R = |emptyset|), so we can
set a new random value for pF (for uniqueness, which might be useful later),
and set pmId = |emptyset| too.

These references, of course, must be authenticated by the operation initiator.
If G supports "additional authenticated data", we can simply use this feature.
Otherwise, perhaps such a feature can be added. For example, some protocols
retroactively authenticate the protocol version to prevent downgrade attacks;
the same mechanism could be adapted to authenticate arbitrary data. Typically
this is not private, but as argued elsewhere, our scheme does not try to
protect the privacy of the existence of session membership changes, and these
references are meant to be hashes that don't leak information about the content
of the session.

If it is not feasible to achieve in-operation authentication, a fallback is to
resend this information as part of a message in the newly-created authenticated
session. Others should expect this message, abort the session if it is not
received within some timeout, or verify it against the older unauthenticated
claims if it is received. This keeps the membership operation component more
decoupled from the rest of the system. However, it would take longer to achieve
our desired security property.

.. [#Npctx] Another option is to refer to earlier context, from when they were
    last in the session. There are many issues to be researched here, e.g. what
    if they were previous in the session, but the inviter does not know this.

Agreement
---------

Now that we have context preserving pI and pF packets, we need a way select a
subset of these that forms a total order. [#Ncord]_ Equivalently, we need an
agreement algorithm, consistent across all members, to accept a single pI/pF
that follows a pF/pI, and reject other proposals.

.. digraph:: hybrid_agreement

    rankdir=RL;
    node [height=0.5, width=0.5, style=filled, label=""];
    i1a [fillcolor="#66ff66", label="+a"];
    f1a [fillcolor="#ff6666", label="s"];
    f1b [fillcolor="#66ff66", label="f"];
    f1c [fillcolor="#ff6666", label="f"];
    i2a [fillcolor="#ff6666", label="+b"];
    i2bf [fillcolor="#66ff66", label="-c,s"];
    i3a [fillcolor="#66ff66", label="+b"];
    f2a [fillcolor="#ff6666", label="f"];
    f1a -> i1a;
    f1b -> i1a;
    f1c -> i1a;
    i2a -> f1b;
    i2bf -> f1b;
    f2a -> i2bf;
    i3a -> i2bf;

The above diagram helps visualise this. Nodes labelled "-m" or "+m" are pI
packets that include or exclude a member from the session; and those labelled
"s" or "f" are success or failure pF packets linked to that operation. Green
nodes are accepted proposals, and all other nodes are red, meaning a rejected
proposal. For every accepted proposal, there is at most one accepted proposal
that references it. In the above example, a proposal to include *a* is accepted
but the operation fails, and is followed by a proposal to exclude *c* which is
accepted and succeeds immediately without requiring more packets; further
proposals to fail "-c" are rejected, and then a second proposal to include *b*
is accepted after the first one was rejected earlier.

Our agreement algorithm, ignoring a few special cases mentioned later, is as
follows: for a given parent pF/pI, we accept the first pI/pF proposal in the
channel event sequence that references it, except that:

Rule XP:
  We ignore proposals that are added to the channel when the channel membership
  does not include its target membership, i.e. if M |NotSubsetEqual| mem[i].

This rule exists because we want everyone in M to reach the same answer. This
is not possible if some of them are not in the channel when the proposals are
echoed back. Given this rule, it is therefore wise, *before* one sends out a
proposal at index i, to check that M |subseteq| mem[i] - though of course this
does not guarantee that this is actually added to the channel at an index j
where M |subseteq| mem[j].

We don't worry about non-(initial or final-candidate) packets - if any are
missing from the channel, then G itself decides whether to finish or not.
Everyone should reach the same conclusion, as long as the transport works
according to our expectations.

This should also work in the degenerate case of a 1-packet operation (e.g. with
a key dictator). Here, a packet is both a pI proposal and a pF that is accepted
immediately, before futher channel events are processed. Implementors may need
to handle this specifically; but this should not cause major difficulties.

The advantage of this agreement algorithm is that it requires no further round
trips between members, beyond what was going to be sent by G anyway. Members
must wait one round-trip before they know their proposals are accepted, but
this is necessary for any total order that preserves context.

One might argue that, if agreement isn't reached, then our consistency checks
below will emit warnings already - so this complexity is unnecessary. However,
those checks are meant to detect *incorrect behaviour* of the transport; but
concurrency conflicts can happen *even when* the transport behaves as expected.
This agreement scheme aims to reduce *false negatives*, which is vital both for
a good user experience, and to avoid training users to ignore warnings.

One attack that a server can execute is to block operations without detection.
When the victim sends a proposal, the server first passes it out-of-band to a
co-operating insider who generates a conflicting proposal. Then, it echoes this
proposal before the victim's, causing members to reject the latter. This is all
within the bounds of normal behaviour of the group transport, so the attack is
not detectable and therefore a problem. For now, we ignore this, since this
power is inherent to the idea of a server-dictated total order, and the need to
have a co-operating insider increases the cost of this attack. Nevertheless,
this is not ideal, and we welcome suggestions for improvements.

.. [#Ncord] That is, over the context-preserving ordering relation, based on
    explicit references, not the group channel ordering relation.

Consistency
-----------

As each member accepts proposals, they calculate a "chain hash" (CH) for it:

pId:
  H(packet data||sender||recipients) \
CH(pId):
  H(CH(pId of previous accepted proposal) || pId || pId-type)

We include the (unauthenticated) channel sender and recipients in the hash for
the packet-id, so that our consistency checks also cover this information. This
is necessary, because our agreement algorithm uses that information. If we omit
them, then the transport could claim different memberships to different
members, which may cause them to reach different results when running the
algorithm, but still calculate the same packet-id.

The CH of an accepted proposal packet, as calculated by a member, attests to
all proposals accepted by them up to and including it. "pId-type" attests to
how they interpret the packet, in case this might be different across members
depending on local state, environment, etc. Here, we use "1" for pI, "2" for
pF, and "3" for pI+pF packets.

As with other things, these may only be calculated when a packet is *received*
from the channel. Further, since packet contents are unique, these packet-ids
are also unique. [#Nhash]_ We ignore rejected packets, because we don't use
information contained in them to make decisions; and this also makes things
easier later when we deal with partial visibility issues.

For incoming new members to be able to match up with everyone's CH values,
every pI should also contain:

prev_CH:
  chain hash of prev_pF = CH(prev_pF)

When an incoming member accepts their first pF, they may read the previous CH
from this value, trusting it opportunistically until the next consistency
check. When starting a new session from scratch (so that there is no prev_pF),
we may use the random prev_pF as mentioned in :ref:`hybrid-context-preserve`;
this should not actually need any specific code to "just work".

Periodically, everyone authenticates and sends the following information:

    ( last pId seen, CH(pId) )

This could be done as part of a message in our (authenticated) causal order
history graph. This essentially corresponds to an "ack" that we described
previously, but for the total order of accepted proposals. Then, we may ensure
consistency by checking that we receive full-acks for every accepted proposal,
similar to as described in :doc:`../causal/02-consistency`.

Why is this mechanism necessary? Surely G already authenticates its result
identically across all members? Well, firstly, we don't assume that G does this
by *all members*, only by some party that is satisfactory. Secondly, *even if*
G does authenticate the result (of a particular pF) by all members, whether it
is actually accepted by each member, is outside of its control. For example,
two members may be able to send two different pF proposals, that can each cause
G to reach different but valid results. With the help of the transport, this
could be arranged, causing the session to split outside of G's control. We
could specify that G should prevent this, but the property is quite hard to
define formally. [#Ngcon]_ So instead we specify this consistency check, that
works regardless of the guarantees that G makes.

.. [#Nhash] Our definition allows an outside observer to potentially calculate
    pIds and CHs. This is not a problem for anything we describe, but may cause
    a problem for applications that build on top of us. One may use a different
    definition, but it would need to be done based on how packets are encoded,
    which is specific to the membership operation. If it is necessary, one may
    adapt :ref:`encoding-message-identifiers` to apply instead to this context.

.. [#Ngcon] We would have to make some statement about the second-to-final
    packet, and how it must depend on all members, or something like that.

.. _hybrid-partial-vis:

Partial visibility
==================

Of initial proposals
--------------------

Earlier, we required that pI proposals must be identifiable (decodable,
interpretable) by all of M'\|M (i.e. using any secret information they have) or
everyone (i.e. without any secret information). Let's look at this in more
detail. We consider the following cases:

1. for all pI with a given prev_pF: only { all of M' } can identify that pI
2. for all pI with a given prev_pF: only { all of M } can identify that pI
3. for all pI with a given prev_pF: only { all of M'\|M } can identify that pI
4. for all pI with a given prev_pF: everyone can identify that pI

M' is specific to the prev_pF and constant across all pI, and M is specific to
each pI. Other cases are too complex to think about, or trivially useless.

With (1), the included members, I := M-M' cannot run the agreement algorithm
and therefore we require further packets to tell them that they are actually
supposed to be participating in G. So we forbid this possibility, because we
are aiming for something with no extra packets. Generally, existing instances
of G don't hide pI from I anyway, so we don't lose much with this.

With (2), different sets of pI (with the same prev_pF) are visible to different
members, since each pI proposes different X := M'-M. It does not seem feasible
to be able to tweak the agreement algorithm to work around this, since there is
no set of members-who-can-see-a-pI that is constant across all these pI. For
example, the auto-kick solution suggested in (3) fails because of the lack of a
constant M' that is able to derive the same actions to execute. So we forbid
this possibility as well. TODO: add the session-split example.

With (3), different sets of pI (with the same prev_pF) are visible to different
members, since each pI proposes different M. So (across all these pI) different
members would reach different results for the agreement. To work around this,
we specify that in this case, after a given pI is accepted by M', they must all
try to kick extra members not in M, and these latter members should interpret
this as "another proposal, with the same prev_pF, for a G that does not include
us, was accepted". Unlike in (2), the M' are constant (being determined by the
same pF), so this is consistent across all members.

With (4), this is the simplest case, and we don't need to handle it specially,
but it leaks information about session membership changes. However, arguably,
in the context of our scheme, this does not cause *extra leakage* - the server
has this metadata anyway, in the form of channel membership changes. If we
require membership changes to be private, we will need to achieve this some
other way, i.e. not using a hybrid ordering on top of a server transport.
(TODO: what about group channels that fit our model, but not using a server?)

Of final proposals
------------------

For the case of pF proposals, we have the similar cases as above, with slightly
different arrangements of quantifiers. For all pF with a given prev_pI, one of
the following might apply:

1. only { all of M' } can identify that pF
2. only { all of M } can identify that pF
3. only { all of M'\|M } can identify that pF
4. everyone can identify that pF

We can immediately forbid (1) for the same reason as in the previous section,
and forbid (4) as being unnecessary - nobody outside of M'\|M needs to identify
pF packets, only pI ones, as per `hybrid-own-enter-channel` - it gives us no
advantage to (3), and we leak less information that way.

For a success pF, it is reasonable and common that G has the security property
that only M may verify (and therefore identify) its final packets. In concrete
terms, flipping a bit to turn a valid final packet into an invalid one, may be
indistinguishable to anyone not in M. This is case (2), so we must support it.

For a failure pF, M' *must* be able to identify pF, since they remain in the
session and will need to refer to this in any subsequent pI proposals. Note
that this implies that we forbid "implicit failures" - e.g. in the "bit flip"
scenario mentioned above, an invalid final packet *must not* be accepted as a
failure pF, because it is not identifiable by X := M'-M. Instead, it should be
ignored, and a *separate* failure pF proposal be sent.

Since different pF are identifiable by different members, we must extend our
agreement algorithm to ensure everyone still reaches the same conclusion. The
extension is similar to the one from (3) in the previous section. If a success
pF is accepted by M, they must all try to kick X, and X should interpret this
as "G was completed to exclude us". This deals with the case where X receives a
failure pF after a success pF (that they can't identify), which may cause them
to incorrectly conclude that G failed.

Session and channel interactions
================================

Entering a channel
------------------

.. _hybrid-own-enter-channel:

The agreement algorithm described so far (including extensions) for a pI
assumes that everyone knows the full event sequence from its prev_pF up to it.
However, this is not the case for new members that entered a channel after the
prev_pF - they don't know if other proposals were sent before they entered.

As a new member entering the channel, the next pI proposal we see, if it is not
already ignored due to rule XP, may be divided into a few cases:

Rule EIO:
  It isn't trying to include us. Then, we ignore it, but assume that others
  will accept it or have already accepted another packet with the same prev_pF.
  So we store prev_pF and reject any future proposals that reference it.

Rule EII:
  It is trying to include us, and its prev_pF has not yet been blacklisted by
  rule EIO. Then, we accept it opportunistically:

  - If we have seen prev_pF, then we know this acceptance is correct.
  - Otherwise, this could be incorrect - another proposal that references the
    same prev_pF could have been accepted before we entered the channel. But in
    this case, the server-order consistency check would fail later, or (if G is
    contributive) we would not be able to complete the operation, or someone
    would kick us because we're in the channel at an inappropriate time.

For others that enter the channel:

Rule ES:
  If another member enters outside of a membership operation, notify the local
  user that they would like to join the session.

Rule EO:
  If another member enters during a membership operation, then (by rule XP)
  they are not part of the pending new sub-session, so kick them. Optionally,
  notify the local user that they probably want to join the channel, so that
  they can manually invite them.

Leaving a channel
-----------------

If we leave a channel for whatever reason, we can no longer be sure that we
didn't miss any packets. Therefore:

Rule LI:
  If we leave the channel, we must clear all state to do with the session. This
  is simpler to reason about than trying some complicated recovery logic,
  especially in terms general to *any* membership operation.

For others that leave the channel:

Rule LS:
  If another member leaves outside of a membership operation, propose an
  operation to exclude them.

Rule LOI:
  If another member leaves during an operation that involves including or
  keeping them in the session, then this is interpreted as a pseudo pF packet
  which is immediately accepted as a *failure* of the operation - since we know
  that the leaver is unable to complete it, having cleared all state.

  The pId of this pseudo-packet is defined as:

    | pId = H(prev_pI||"leave"||remaining channel members)

Rule LOX:
  If another member leaves during an operation that involves excluding or
  keeping them in the session, and it fails (for the latter case, this is
  inevitable due to rule LOI), then propose an operation to exclude them.

The following rules are about entering, but the reasoning behind them demands
that we discuss them *after* the above rules about leaving:

Rule EAL:
  When trying to exclude a member from the cryptographic session, who already
  left the channel: if this member re-enters the channel before we exclude them
  from the session, we can just auto-kick them until we exclude them.

The above auto-exclusion rules, together with rule XP, should ensure that we
mostly don't get to a state where a member is in the channel and expects to be
included, but no-one tries to do this.

If everyone that wants to exclude them leaves the channel, then there are a few
race conditions where they would never be kicked. [#Nrace]_ So, they should
leave and re-enter the channel after a timeout. When they leave, existing
members will notice it and later try to exclude them, which is the corrected
behaviour that we want.

.. [#Nrace] For example:

  1. X leaves and re-enters. X is still part of others' sessions, but X has
     cleared that session (rule LI) and expects to be included into a new one.
  2. G accepted to add Y, G completes, Y now part of session and thinks X is.
  3. X doesn't get auto-kicked for whatever reason (e.g. slow transport)
  4. Everyone that wants to exclude X leaves (e.g. disconnected), but X remains
     in the channel.
  5. Y remains in session, X still in Y's old session, but (as in step 1)
     expects to be included into a new session.

Excluded from a session
-----------------------

When we exclude someone, as per previous sections, they stay in the channel
until they are kicked, since they can't identify a success pF. This means that
they have state inconsistent with other members, so we can't re-include them,
until they've left the channel and reset their state. In other words:

Rule IAL:
  When someone tries to include a member into the session, who was previously
  excluded but not yet left the channel, auto-kick them from the channel.

If everyone that wants to kick them leaves (e.g. disconnected), then they would
never be kicked. So, they should auto-leave the channel after a timeout, in
case this happens, ideally after consistency is reached or itself times out.
