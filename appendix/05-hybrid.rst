===============
Hybrid ordering
===============

Causal orders have some nice properties explained elsewhere, but are also hard
to implement correctly when not everyone is allowed to see the full history.
(At the time of writing, we don't have a full solution for this yet.)

One alternative option is to have a total order for membership operations,
while leaving messages to be in a causal order. As :doc:`discussed earlier
<../causal/index>`, a total order is probably only viable in a low-latency
synchronous application, such as instant messaging. But the concepts involved
are less advanced, so more straightforward to implement with current tools.

Consistent server order
=======================

We propose one fairly simple way of achieving this, that requires a server.
This is quite standard for existing instant messaging protocols such as XMPP,
and requires no extra functionality on the server side. Therefore, clients can
transparently run this scheme on top of existing infrastructure.

We expect that the server echoes back packets to all clients in the same total
order, which is the case for the aforementioned existing protocols. [#Npkt]_ We
do not assume that the server is honest: we outline a scheme where clients can
verify this property. For each member, as they receive interesting packets from
the server, they calculate a "chain hash" (CH) for it:

    | pId = H(packet) \
    | CH(pId) = H(CH(previous pId or "") || pId)

By "interesting", we mean packets that we care about verifying the consistency
of the server order of. We can ignore other packets, such as message packets
(which are already authenticated and consistent via other mechanisms described
earlier) and junk or resent packets. "Interesting" will be defined in more
detail later, but includes at least "initiate operation" packets. From here on,
we'll just refer to these as "packet".

We assume that packets are unique, so that our hash-defined packet ids are also
unique. The membership operation, and anything else that is "interesting", must
satisfy this property. This should not be difficult; indeed otherwise it would
be susceptible to a replay attack. Duplicate packets should be ignored.

The CH of a packet, as calculated by a member, attests to the server order of
all older packets up to and including it, that they received from the server.
Note that even the author of the packet cannot calculate its CH until they
receive it back from the server, since they may receive other packets between
when they send and receive it.

Periodically, everyone authenticates and sends the following information:

    { last pId seen: CH(pId) }

It is important that this be authenticated; it could be done as part of a
message in our (authenticated) causal order history graph.

Then, we may verify consistency by doing something similar to the mechanism
described in :doc:`../causal/02-consistency`, though the book-keeping data
structures can be simpler. That is, instead of keeping a "not yet acked" set of
members for every packet, we keep a "last pId acked" value for every other
member. When we receive the above information from someone, we check their CH
against ours, and if it's correct we update the "last pId acked" value we have
stored for them (perhaps checking that it's not older than the previous value).
If the CH values don't match, we abort. If any "last pId acked" value does not
match our own "last pId seen" value after some timeout, we emit a "not
fully-acked" warning to the user.

When a new member is included into the session, the first (interesting) packet
that is visible to them, should include the CH of the last packet not visible
to them. This information does not need to be authenticated; it is only used by
the new member to calculate its own CHs, which they use as per above to verify
the server order consistency - which *is* authenticated.

(TODO: above needs to be thought through more precisely, e.g. what if they
join during the middle of an existing operation?)

So, we have a verifiable server-dictated total order. This is a topological
ordering of the underlying partial order of packets. It does *not* preserve
context - when we send a packet out, we have some context, but the server may
send us (and others) more packets between this context and our packet. (TODO:
draw a diagram for this.) However, we can use this order as a tool to build a
total order that *does* preserve context, at least for membership operations.

.. [#Npkt] We use the term "packet" to refer to transport-level packets; the
    term "message" refers specifically to verified-decrypted messages, that
    comprise the causal order history graph.

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

To ensure that the server doesn't reorder membership operations, any pI must
reference the pF from the previous operation (or null if it's the first). This
information must be authenticated somehow: maybe in a future message (simpler
to implement, but slower to run and requires timeout semantics) or as part of
the operation's internal authentication (faster to run, but more invasive to
implement, maybe complex).

There may be concurrent multiple different pI packets that point to the same
pF. The server order can "break ties" between these - for all pI that point
to a given pF (or null), only the first one counts, and members ignore/reject
every other such pI. (They must remain part of the server order, however.)

Likewise, if a membership operation is taking too long (e.g. maybe someone has
gone offline, so it will never finish) then an existing member may choose to
send an "abort" packet. Concurrently, someone may send a pF that completes the
operation; or someone may issue an "fail" packet, if the operation supports
that. Here again, the server order breaks ties.

So, a minimal list of "interesting" packets for which we must verify server
order for (see previous section) are:

- pI: initiate
- pF: complete (aka finish with success)
- pF: fail (aka finish with error)
- pF: abort

One attack the server can execute in this position is to block operations
"innocently". For example, when victim V sends a pI, the server first passes
it out-of-band to a co-operating insider M who generates a conflicting pI. The
server then broadcasts this conflicting pI before the victim's, negating it
within the bounds of "normal behaviour" as defined by our scheme. This is a
problem because the attack is not detectable. For now however, we'll ignore it,
since this power is inherent to the idea of a server-dictated total order. This
is not ideal of course, and we welcome suggestions for improvements.

So, now we have a context-preserving authenticated session-global total order
of membership operations:

- | by our requirements of the membership operation,
  | every pF is authenticated and linked to some earlier pI
- | by the server order,
  | every pF is unique for the pI it is linked to - others are rejected
- | by our requirements of implementations of our scheme,
  | every pI is authenticated and linked to some earlier pF (or "null")
- | by the server order,
  | every pI is unique for the pF it is linked to - others are rejected

(TODO: link these with the message partial order graph)

.. [#Ncon] Or rather, they *will have* confidence, since consistency checks
    inherently must occur *after* the packet has been received and processed by
    the component that executes membership operations.

.. [#Nack] Note the similarity in reasoning on why :ref:`we must ack messages
    ourselves <full-ack>`.
