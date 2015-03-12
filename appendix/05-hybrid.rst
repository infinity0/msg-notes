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
detail later, but includes at least "initiate key agreement" packets. From here
on, we'll just refer to these as "packet".

We assume that packets are unique, so that our hash-defined packet ids are also
unique. The key agreement, and anything else that is "interesting", must
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

TODO

- abstract model of a key agreement with first and last packet
- propose/accept/reject based on server order
- KA propose, KA complete/cancel/error packet types
