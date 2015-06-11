===============
Hybrid ordering
===============

Causal orders have some nice properties explained elsewhere, but are also hard
to implement correctly when not everyone is allowed to see the full history.
(At the time of writing, we don't have a full solution for this yet.)

One alternative is to have a total order for membership operations, while
leaving messages to be in a causal order. Here, we present a scheme for this
that requires no extra client packets, but does require one extra round of
latency - the minimum necessary for a total order that preserves context. Note
however that this is strictly necessary only for the membership operations - so
perhaps an intelligent application can use it even for asynchronous messaging
during periods of static membership.

Consistent server order
=======================

We expect a server that receives all packets [#Npkt]_ (join, part, message)
from all clients, and echoes them back reliably to everyone in the same total
order. This is quite standard for existing instant messaging protocols such as
XMPP, and requires no extra logic on the server side. Therefore, clients can
transparently run this scheme on top of existing infrastructure.

We do not assume that the server is honest or competent about this: we outline
a scheme where clients can verify this property. Every member, as they receive
interesting packets from the server, they calculate a "chain hash" (CH) for it:

    | pId = H(packet data||sender||recipients) \
    | CH(pId) = H(CH(previous pId or "") || pId || pId-type)

By "interesting", we mean packets that we care about verifying the consistency
of the server order of. We can ignore other packets, such as message packets
(which are already authenticated and consistent via other mechanisms described
earlier), rejected packets, and junk or resent packets. "Interesting" will be
defined in more detail later. (It should be clear from context whether we are
talking about packets or only "interesting" packets.)

We include the (unauthenticated) author and recipients in the hash, which means
that our consistency checks also commit to the consistency of this metadata at
those packets. This is important, because our logic about which packets to
accept/reject also depends on this information (as described later).

pId-type is an optional piece of information, if we want to also check that
everyone interprets the packet in the same way, in case this might be ambiguous
depending on local state, environment, etc.

We assume that packets are unique, so that our hash-defined packet ids are also
unique. The membership operation, and anything else that is "interesting", must
satisfy this property. This should not be difficult; indeed otherwise it would
be susceptible to replay attacks. Duplicate packets should be ignored. [#Nhsh]_

The CH of a packet, as calculated by a member, attests to the server order of
all older packets up to and including it, that they received from the server.
Note that even the author of the packet cannot calculate its CH until they
receive it back from the server, since they may receive other packets between
when they send and receive it.

Periodically, everyone authenticates and sends the following information:

    ( last pId seen, CH(pId) )

It is important that this be authenticated; it could be done as part of a
message in our (authenticated) causal order history graph. Then, we may verify
consistency by doing something similar to the mechanism described in
:doc:`../causal/02-consistency`.

So, we have a verifiable server-dictated total order. This is a topological
ordering of the underlying partial order of packets. It does *not* preserve
context - when we send a packet out, we have some context, but the server may
send us (and others) more packets between this context and our packet. (TODO:
draw a diagram for this.) However, we can use this order as a tool to build a
total order that *does* preserve context, at least for membership operations.

.. [#Npkt] We use the term "packet" to refer to transport-level packets; the
    term "message" refers specifically to verified-decrypted messages, that
    comprise the causal order history graph.

.. [#Nhsh] Our definition allows an outside observer to potentially calculate
    pIds and CHs. This is not a problem for anything we describe, but may cause
    a problem for applications that build on top of us. One may use a different
    definition, but it would need to be done based on how packets are encoded,
    which is specific to the membership operation. If it is necessary, one may
    adapt :ref:`encoding-message-identifiers` to apply instead to this context.

Context-preserving total order
==============================

We'll abstract out the idea of a membership operation as much possible - so the
implementation could be a group key agreement, a dictator handing out a key,
etc. However, it must satisfy the following:

We assume that the membership operation handles internal authentication and
ordering of its own packets. Packets cannot be replayed and misinterpreted to
be part of a different operation. If the operation finishes (with success or
failure) or aborts early, then this action is authentic. If it successfully
finishes ("completes"), then any resulting information is also authentic.

We also assume that the operation is initiated and finished in-band, in the
server order. This means there must be a precise and well-defined first packet
pI, that the initiator sends, and last packet pF, that the server may get to
ultimately decide, if e.g. concurrent packets in a "final round" could validly
be reordered without intefering with the execution of the operation.

When everyone, except the sender of pF, receives pF, they are able to locally
complete the operation, and also have confidence [#Ncon]_ that everyone else
has done so because of the consistency of the server order. For the sender of
pF, they may be able to complete the operation when they receive the packet
*preceding* pF, but they must, like the others, wait until they receive pF
back, to have confidence that everyone else has also completed the operation.
[#Nack]_ In either case, it should be obvious to everyone how to identify pF,
by looking at the server order.

Demonstrably, pF comes after pI, both in the server order and from the point
of view of its sender (i.e. context-preserving). We'll now outline a scheme
that extends this across operations, and also handles failures and aborts.

To ensure that the server doesn't reorder operations, any pI proposal must
reference the pF from the previous operation (the next section deals with the
case if there was no previous). It should also reference the latest messages in
the (partially-ordered) ongoing session derived from pF. This information must
be authenticated eventually; we go into strategies for this further below.

There may be concurrent multiple different pI proposals that reference the same
pF. We use the server order to "break ties" between these - for all proposals
that point to a given pF, only the first one counts, and members ignore/reject
every other such proposal.

Likewise, if a membership operation is taking too long (e.g. maybe someone has
gone offline, so it will never finish) then an existing member may propose to
send an "abort" packet. Concurrently, someone may propose a pF that completes
the operation; or someone may propose a "fail", if the operation supports that.
Here again, the server order breaks ties.

If the membership operation supports outsider initiation (i.e. *join* as well
as *invite*), it should be the reply packet that is treated as the pI within
our scheme here. This is authored by an insider who knows the CH of the last
packet, which (as above) should be included in the reply too so the new member
can verify the server order consistency. As previously, ties between concurrent
reply proposals are broken by server-order.

One attack the server can execute here is to block operations "innocently". For
example, when victim V sends a pI, the server first passes it out-of-band to a
co-operating insider M who generates a conflicting pI. Then, it broadcasts this
conflicting pI before the victim's, negating it within the bounds of "normal
behaviour" as defined by our scheme. This is a problem because the attack is
not detectable. For now however, we'll ignore it, since this power is inherent
to the idea of a server-dictated total order. This is not ideal of course, and
we welcome suggestions for improvements.

Note that this scheme also works in the degenerate case of a 1-packet operation
(e.g. with a key dictator) - in this case, a packet may be both a pI proposal
and a pF that is accepted immediately. Implementors should check for this case
if appropriate, by immediately trying to decode a pI packet as a pF packet if
the former is accepted.

So, in this context, "interesting" packets for which we must verify server
order for (see previous section) are accepted pI and pF proposals. As for the
pId-type we mentioned as a way to also commit the "interpretation" of packets
into the chain hash consistency check, we'll use "1" for pI, "2" for pF, and
"3" for pI+pF packets. We don't include rejected packets in this definition,
because they are redundant; and actually this makes things easier later when we
run into partial visibility issues.

Every pI proposal must contain the following information:

- last accepted pF, to preserve the author's context
- CH(pF), for new members to verify server-order consistency
- latest messages seen, in the ongoing session derived from pF

This information must be authenticated. If the membership operation supports
"additional authenticated data", we can simply use this feature. Otherwise,
perhaps such a feature can be added. For example, some existing protocols
retroactively authenticate the protocol version to prevent downgrade attacks;
the same mechanism could be adapted to authenticate arbitrary data. Typically
this is not private; but as argued below, neither does the scheme we present
here try to protect the privacy of group membership changes.

If it is not feasible to achieve in-operation authentication, a fallback is to
resend this information as part of a message in the newly-created authenticated
session. Others should expect this message, abort the session if it is not
received within some timeout, or verify it against the older unauthenticated
claims if it is received. This keeps the membership operation component more
decoupled from the rest of the system. However, it would take longer to achieve
our desired security property.

So, now we have a context-preserving authenticated session-global total order
of membership operations:

- | by our requirements of the membership operation,
  | every pF is authenticated and linked to some earlier pI
- | by the server order,
  | every pF is unique for the pI it is linked to - others are rejected
- | by our requirements of implementations of our scheme,
  | every pI is authenticated and linked to some earlier pF
- | by the server order,
  | every pI is unique for the pF it is linked to - others are rejected

.. [#Ncon] Or rather, they *will have* confidence, since consistency checks
    inherently must occur *after* the packet has been received and processed by
    the component that executes membership operations.

.. [#Nack] Note the similarity in reasoning on why :ref:`we must ack messages
    ourselves <full-ack>`.

Corner cases caused by partial visibility
=========================================

Partial visibility causes some corner cases here too. To start off with, we'll
clarify some of our assumptions on the membership operation and visibility.

We require that all pI proposals must be identifiable and decodable by all
channel members, even those not part of the cryptographic session. We feel this
is necessary; the alternatives seem much more complex:

- If they are only identifiable by current session members, then new members
  cannot be part of the acceptance process and must be told explictly which one
  was accepted. This requires further packets, but the operation may complete
  (or even have further operations accepted) concurrently in the meantime.

- If they are only identifiable by current members and the specific new members
  they are including, then different proposals on top of the same prev_pF would
  be visible to different members. Then, we also require further packets to
  reach an agreement on what was accepted.

The server has this metadata anyway. If members require session membership
changes to be private, we will need to achieve this some other way, i.e. not
using a hybrid ordering on top of a server transport.

However, we do assume that we may not be able to identify all pF proposals for
operations not involving us. Even if failure proposals are visible, success
proposals may not be, since these could depend on the cryptographic state of
the group. This means we may have uncertainty about which pF proposal was
accepted. This causes some more complexity, but is easier to work with.

Leaving a channel
-----------------

If we leave a channel for whatever reason, we can no longer be sure that we
didn't miss any packets. Therefore, we should clear all state to do with the
session. (This is simpler to reason about than trying some complicated recovery
logic, especially in terms general to *any* membership operation.) Given this,
we add another rule:

- (rule XO) if any member leaves during an operation that involves including or
  keeping them in the session, then this is interpreted as a pseudo pF packet
  which is immediately accepted as a *failure* of the operation - since we know
  that the leaving member is unable to complete it, having cleared all state.

The pId of this pseudo-packet is defined as:

    | pId = H(prev_pI||"leave"||remaining channel members)

TODO(xl): what if the leaving member re-enters before we exclude them?

Members leave during proposal echo delay
----------------------------------------

When we send a pI proposal, even if at this time the new members are in the
channel, we may see them leave the channel before our proposal is echoed back.
So, we add an additional check:

- (rule XP) if any pI proposal is received back from the server at a point when
  any of its new members (i.e. the membership for the new sub-session it is
  proposing) are not in the channel, then this proposal is rejected, even if it
  would otherwise be accepted.

- if any pF proposal is received under similar circumstances, the operation
  should already have been failed as per the previous section - so this case
  doesn't need explicit logic to cover

The membership of the channel should be obvious to everyone in the channel,
assuming the server echoes back membership events in the same order, which (as
described above) is checked retroactivly.

Given the above checks, it is therefore wise for proposers to make sure new
members are in the channel *before* they send out their pI packet.

Entering a channel
------------------

The scheme described so far lets members agree on which pI proposal to accept,
if they know the full server order from its prev_pF up to it. However, this is
not the case for new members that entered a channel after the prev_pF - they
don't know if other proposals were sent before they entered.

The next pI proposal we see, may be treated by a few cases:

- It is invalidated by (rule XP). Then, we know others would ignore it, so we
  should ignore it too. Otherwise:
- (rule IO) It isn't trying to include us. Then, we ignore it, but assume that
  others would either accept it, or have already accepted another packet that
  references the same prev_pF. Either way, we store prev_pF and reject any
  future proposals that reference it.
- (rule II) It is trying to include us, and it's not invalidated by rule IO.
  Then, we accept it opportunistically:

  - If we have seen prev_pF, then we know this is correct.
  - Otherwise, this could be incorrect - another proposal that references the
    same prev_pF could have been accepted before we entered the channel. But in
    this case, the server-order consistency check would fail later, or (if it
    is contributive) we would not be able to complete the operation, or someone
    would kick us because we're in the channel at an inappropriate time.

There are lots of failure modes; members could lie about what their prev_pF is,
or the server could manipulate this value, etc. Rather than trying to enumerate
and handle them all, we just depend on our server-order consistency mechanism,
and issue an error after a timeout if this isn't reached. This is why adding
only accepted packets into our CH is better than also adding rejected packets
too - the CH also "commits" to which packets we accepted. If we also added
rejected packets here, we could have some failure modes where members accepted
the same proposals yet have different CH values.

To clarify, what this scheme does is to allow the protocol to work succesfully
under *innocent* race conditions caused by asynchronity and partial visibility.
This reduces the amount of false positives (in terms of failures of security
checks), making the overall system more user friendly. As with any secure
system, attacks cause true positive failures.

TODO: iron out the case of what to do when joining a new channel with no
existing session. Roughly along the lines of, locally generate random prev_pF.

Failure of others to exclude self
---------------------------------

When we are being excluded but this fails, we should be able to remain in both
the channel and session, and stay consistent with everyone. By our assumptions,
we may not be able to identify which pF proposal was accepted. However, if it
succeeds then we'll get kicked and know to close the session and clear all
state. So, we can opportunistically assume that the operation will fail and
continue participating, until someone kicks us from the channel.

Responding to channel membership changes
----------------------------------------

When a greeting is in progress (between an accepted pI and an accepted pF):

- If someone leaves, or was forced to leave:

    - If they were meant to be part of the new sub-session, fail the operation (rule XO)
    - Else, they are supposed to leave; do nothing

- If someone enters, or was forced to enter:

    - By (rule XP) they are not needed in the new sub-session, so kick them.
      If the application wishes to support spontaneous requests-to-join from
      outsiders, then perhaps notify the local user about this.

Outside of a greeting:

- If someone leaves, propose an operation to exclude them.
- If someone is forced to leave, assume someone else will make such a proposal.
- If someone enters, notify the user that they would like to join the session.
- If someone is forced to enter, assume someone else will make such a proposal.

