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
Define a message m to be *fully-acked* iff |forall| r |in| recipients(m): m
is acked-by r. Both of these depend implicitly on the current transcript.
[#Nvis]_ Consistent with this definition, a user would consider m to be
unacked by *themselves*, if they haven't yet sent a message that refers to
it - this is necessary for *others* to know we have seen the message. Note
that in our terms, an "ack" is simply any message that refers to a previous
one (the one being acked), not necessarily a special ACK message.

Once a message is acked by everyone else, we are certain that they have seen
it, and we reach message consistency. If this does not happen after a grace
period, we should alert the user. On top of this, we can resend the message.
This should be the exact same ciphertext as was sent before - this makes it
easy for the recipients to resolve any duplicates, and also allows us to
resend a message originally by someone else, if deemed beneficial. This
general approach of pro-active resends ought to greatly improve performance
under bad network conditions. However, optimising the exact policy for it
can get very complex so we'll skip that discussion for now.

TODO: draw some diagrams
TODO: introduce the term "ack-monitor"

(The reason we define *fully-acked* is that it is the state we want to
eventually reach - everyone else is expecting an ack from us too. Given
other considerations to be introduced later, it is simpler to cancel our
ack-monitor when a message is fully-acked, rather than when it is only
acked-by-others.)

These strategies may be done incrementally, instead of having to wait until
the end of the session. A further advantage is that there is no bandwidth
overhead - we make use of the parent references we are already sending.

An implementation should probably track unacked recipients, rather than
acked ones: when delivered, each message has a set unackby(m) that starts
off equal to recipients(m), and gradually becomes empty as later messages
are delivered. When the set becomes empty, its ack-monitor is cancelled and
a "fully-acked" event is emitted.

It is also useful to track a set unacked() of messages not-yet fully-acked
in the current transcript. When a message m is delivered, it is added to
this set, then removed again when unackby(m) becomes empty.

.. [#Nvis] This definition becomes slightly more complex when we introduce
    partial visibility; see that chapter for details.

Implicit vs explicit
--------------------

At any given point, the latest messages will not have been fully-acked. To
end the session, or during a lull in the conversation, we therefore need to
send explicit auto-acks. (Of course, we don't need to wait for these acks to
themselves be acked.)

TODO: what to do when the device is away? semantic difference between auto
and manual acks.

Our scheme so far ensures consistency for delivered messages. What about
messages not yet delivered - received messages that sit in the "dangling
messages" buffer for a long time, or messages that we haven't even received?
TODO

TODO:
Dealing with duplicates received: this is because someone thinks we haven't
acked the message yet.

- If we implicitly acked it with a message M, don't need to take any action,
  since we already have a ack-monitor on M that will resend it if necessary.
  Doing a resend now is redundant and may improve latency or clog the network.
- If we explicitly acked it with an auto-ack, then we must resend the same
  auto-ack, since we don't set ack-monitors on auto-acks.

Display
-------
