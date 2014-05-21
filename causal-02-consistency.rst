======================
Transcript consistency
======================

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

One necessary property of a group session is that everyone sees the same
transcript - that is, the same set of messages and their relative orders.
Our causal order already embeds the ordering within each message, so we only
need to ensure consistency of the set of messages.

Acknowledgements
----------------

We start by considering the smaller problem of message consistency. When we
see a message, we want to be sure that everyone else also saw it. There are
many possible reasons why someone might not see a message intended for them,
and often it is innocent. Rather than trying to enumerate and detect all
possible failures, we aim for a certainly-good state. We can be sure that r
saw m, if we see a message by r that refers to m. Conveniently, we already
have the mechanics for this in our causal order, namely the property that
every message refers (transitively) to all its ancestors.

Define a message m to be *acked-by* r iff |exists| m' |in| by(r): m < m'.
Define a message m to be *fully-acked* iff |forall| r |in| recipients(m): m is
acked-by r. [#Nvis]_ Both of these depend implicitly on the current transcript.
Note that in our terms, an "ack" is simply any message that refers to a
previous one (the one being acked), not necessarily a special ACK message.

Once a message is acked by everyone else, we are certain that they have seen
it, and we reach message consistency. If we did not send the message, we must
also make sure that we ack it ourselves  - this is necessary for *others* to
reach message consistency. After this, the message is *fully-acked*, and we no
longer need to worry about it. In a valid graph, a message becomes fully-acked
at the same point for everyone - one can think of the acks as forming a
"reverse" |bot|-less context(m). (TODO: prove equality.)

This is an incremental consistency; full-acks occur as new messages arrive,
instead of all at once at the end of the session. A further advantage is that
there is no bandwidth overhead; we make use of the information we are already
sending.

TODO: draw some diagrams

Between when a message is delivered and is fully-acked, we need a way to check
that it does eventually become fully-acked, and warn or try to recover if this
process takes too long. The mechanism must be pro-active, so it must be outside
of the packet send/recv logic. We'll call this an "ack-monitor", and typically
it would be activated after a grace period after delivery, and de-activated or
cancelled on full-ack.

At the very least, the user should be alerted if message consistency is not
reached. On top of this, we can resend the message. This should be the exact
same ciphertext as was sent before - this makes it easy for the recipients to
resolve any duplicates, and also allows us to resend a message originally by
someone else, if deemed beneficial. This general approach can greatly improve
performance under bad network conditions. However, optimising the exact policy
can get very complex so we'll skip that discussion for now.

.. [#Nvis] This definition becomes slightly more complex when we introduce
    partial visibility; see that chapter for details.

Implicit vs explicit
--------------------

Another thing the ack-monitor should handle is (for others' messages) if we
don't ack the message ourselves. In a busy conversation, the user will likely
send a message within the grace period *anyway*, that will be implictly treated
as an ack. In this case, we don't need to do anything. However, at the end of a
session, or during a lull in the conversation, we will need to automatically
send an explicit ack on behalf of a user.

There are some nuances about this. For example, these auto-acks should not have
ack-monitors registered on themselves, which would result in an infinite loop
of mutual acks (though see the next section for a discussion on heartbeats).
They also affect resend and dedupe logic.

TODO:
Dealing with duplicates received: this is because someone thinks we haven't
acked the message yet.

- If we implicitly acked it with a message M, don't need to take any action,
  since we already have a ack-monitor on M that will resend it if necessary.
  Doing a resend now is redundant and may improve latency or clog the network.
- If we explicitly acked it with an auto-ack, then we must resend the same
  auto-ack, since we don't set ack-monitors on auto-acks.

TODO: what to do when the device is away? semantic difference between auto
and manual acks.

Further issues
--------------

Our scheme so far ensures consistency for delivered messages. What about
messages not yet delivered - received messages that sit in the "dangling
messages" buffer for a long time, or messages that we haven't even received?
TODO

An implementation should probably track unacked recipients, rather than acked
ones: when delivered, each message has a set unackby(m) that starts off equal
to recipients(m), and gradually becomes empty as later messages are delivered.
When the set becomes empty, its ack-monitor is cancelled and a "fully-acked"
event is emitted. It is also useful to track a set unacked() of messages
not-yet fully-acked in the current transcript. When a message m is delivered,
it is added to this set, then removed again when unackby(m) becomes empty.
TODO: maybe move this to an "implementation" appendix.

Display
-------
