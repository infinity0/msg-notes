======================
Transcript consistency
======================

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

One necessary property of a group session is that everyone sees the same
transcript - that is, the same set of messages and their relative orders. Our
causal order already embeds the ordering within each message, so we only need
to ensure consistency of the set of messages.

Acknowledgements
----------------

We start by considering the smaller problem of message consistency. When we see
a message, we want to be sure that everyone else also saw it. There are many
possible reasons why someone might not see a message intended for them, and
often it is innocent. Rather than trying to enumerate and detect all possible
failures, we aim for a certainly-good state. We can be sure that r saw m, if we
see a message by r that refers to m. Conveniently, we already have the
mechanics for this in our causal order, namely the property that every message
refers (transitively) to all its ancestors.

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

Automatic and explicit acks
---------------------------

Another thing the ack-monitor should handle is (for messages sent by others) if
we don't ack the message ourselves. In a busy session, the user will likely
send a message within the grace period *anyway*, that will be implictly treated
as an ack. In this case, we don't need to do anything. However, at the end of a
session, or during a lull in the conversation, we will need to *automatically*
send an *explicit* ack on behalf of a user. This would be a special message
that is not displayed as normal in the user's UI, but is still kept in the
transcript causal order data structure, in order to track full-acks.

There are some nuances about this. The fact the ack is explicit and carries no
other purpose, means that these need not have ack-monitors registered on them.
Indeed, in the automatic case, this would result in an infinite loop of acks
(but see the next section for a discussion on heartbeats).

An implicit ack, such as a normal user message, indicates "some" level [#Nack]_
of understanding of previous messages. Automatic explict acks *should not* be
interpreted to carry this same weight, because the user has no control over
whether they actually read those messages or not. If one desires an explicit
"user-level" ack (e.g. in critical situations) there are a few options:

- a manual explicit ack, that must be initiated by the user - like acks in
  `Pond`_, which are just empty messages.

- a pseudo-manual explicit ack, that may be interpreted like a manual explicit
  ack. This is triggered automatically, but only when the user is interacting
  with the application, or has it focused in the foreground.

These would be implemented as a supplement to the automatic ack. Other
projects' terminology for these concepts include "delivery receipt" for
"automatic ack" and "read receipt" for "manual/pseudo-manual ack".

Explicit acks also affect resend and dedupe logic. When you receive a duplicate
message, this is (in a normal situation) because someone thinks we haven't
acked the message yet.

- If we implicitly acked it with a message m, we don't need to take any action
  - we already have a ack-monitor on m that will resend it if necessary.
  Doing a resend now is redundant and may improve latency or clog the network.
  TODO: rw, with below

- If we explicitly acked it, then we must resend the same ack, since we don't
  set ack-monitors on explicit acks.

TODO: pro-active resends of others' messages: DoS prevention vs recovery from
availability partitions

.. _Pond: https://pond.imperialviolet.org/tech.html

.. [#Nack] The user could avoid reading the messages, but we can't do anything
    about this. In the *average* case, there is some understanding.

Further issues
--------------

Our scheme so far ensures consistency for delivered messages (including
messages we sent). What about messages not yet delivered - e.g. received
messages that sit in the "dangling messages" buffer for a long time, or
messages that we haven't even received?
TODO

An implementation should probably book-keep unacked recipients, instead of acked
ones: when delivered, each message has a set unackby(m) that starts off equal
to recipients(m), and gradually becomes empty as later messages are delivered.
When the set becomes empty, its ack-monitor is cancelled and a "fully-acked"
event is emitted. It is also useful to track a set unacked() of messages
not-yet fully-acked in the current transcript. When a message m is delivered,
it is added to this set, then removed again when unackby(m) becomes empty.
TODO: maybe move this to an "implementation" appendix.

Display
-------
