===========
Consistency
===========

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

Define a message m to be **acked-by** r iff |exists| m' |in| by(r): m < m'.
Define a message m to be **fully-acked** iff |forall| r |in| recipients(m): m
is acked-by r. [#Nvis]_ Both of these depend implicitly on the current
transcript. In our terms, an "ack" is simply any message that refers to a
previous one (the one being acked), not necessarily a special ACK message.

Once a message is acked by everyone else, we are certain that they have seen
it, and we reach message consistency. If we did not send the message, we must
also make sure that we ack it ourselves  - this is necessary for *others* to
reach message consistency. After this, the message is *fully-acked*, and we no
longer need to worry about it. In a valid transcript, a message m becomes
fully-acked at the same point for everyone, namely when a process delivers all
the direct acks { min(by(r) |cap| des(m)) | r |in| recipients(m) }, des(m)
being the descendant messages of m. Notice the similarity in structure to the
definition of context(m) - one may think of these acks as a "future" context(m)
which (if full-ack is reached) contains no |bot| values.

.. digraph:: acks

    rankdir=BT;
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
instead of all at once at the end of the session. This means that the number of
messages whose consistency we are uncertain of, should stay roughly constant
during the lifetime of a normal session. A further advantage is that there is
no bandwidth overhead - we make use of the information we are already sending.

Between when a message is delivered and is fully-acked, we need a way to check
that it does eventually become fully-acked, and warn or try to recover if this
process takes too long. The mechanism must be pro-active, so it must be outside
of the packet send/recv logic. We'll call this an "ack-monitor", and typically
it would be activated after a grace period after delivery, and de-activated or
cancelled on full-ack. [#Nimp]_

At the very least, the user should be alerted if message consistency is not
reached. On top of this, we should resend the message, in anticipation that our
original message was not received for whatever reason. This should be the exact
same ciphertext as was sent before - this makes it easy for the recipients to
resolve any duplicates, and also allows us to resend a message originally by
someone else. This implicit recovery technique results in a simpler protocol
and transcript, without needing an explicit "ensure consistency" message.

Optimising the exact policy of executing resends can get very complex, so we'll
skip that discussion for now. In practise, we have been using an exponential
backoff algorithm, which seems to work adequately.

.. [#Nvis] This definition becomes slightly more complex when we introduce
    partial visibility; see that chapter for details.

.. [#Nimp] An implementation should book-keep unacked recipients instead of
    acked ones: when delivered, each message has a set unackby(m) that starts
    off equal to recipients(m), and gradually becomes empty as later messages
    (that ack it) are delivered. When the set becomes empty, its ack-monitor is
    cancelled and a "fully-acked" event is emitted. It is also useful to track
    a set unacked() of messages not-yet fully-acked in the current transcript.
    When a message m is delivered, it is added to this set, then removed again
    when unackby(m) becomes empty. TODO: maybe move this to an "implementation"
    appendix.

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
Indeed, in the automatic case, this would result in an indefinite sequence of
mutual acks - but see the next section for more discussion on this. However,
ack-monitors for an implicit ack am sent directly after a sequence of explict
acks, should also resend these whenever it resends am - because everyone must
receive anc(am) to be able to deliver am, and we have no other local active
mechanism to resend those explicit acks.

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

Resends and deduplication
-------------------------

When you receive a duplicate message m, this is (in a normal situation) because
someone thinks we haven't acked the message yet - and perhaps we haven't. So we
need to ack it, until we are confident they have received the ack:

- If m is still in the delivery queue, ignore the duplicate. We don't even
  know if it's a valid message yet - we haven't received all its ancestors.
  If and when we do, we would already set an ack-monitor on it, which will
  handle any future duplicates.

- If we haven't yet acked m, the ack-monitor we already set on m will handle
  this situation - i.e. either wait for the user to send an implicit ack, or
  send an explicit ack later.

- If we have already acked m, say with message am, then:

  - If the ack was implicit, then we already set an ack-monitor on am, which
    should handle this situation, resending am if necessary. (Perhaps am was
    already fully-acked, in which case m was sent in error, or due to a heavily
    delayed network, or a replay attack. In any case, the ack-monitor will have
    already been cancelled and we ignore m, which is the correct behaviour.)

  - If the ack was explicit, then we don't have an ack-monitor on am, so we
    should resend am. (This is the only case we take action on.)

Note that we say "confidence", not "guarantee". This is because with explicit
acks, we do not check for full-ack (of the ack) - so perhaps it will be dropped
and the recipient will never see it. (If the network is not dropping messages
though, we will see their resend m, and be able to act on it as above.)

We believe that this is not a security problem - it is the responsibility of
each user to check that all messages they see are fully-acked. If our ack is
dropped, then even though we don't know they have received our ack, we know
that they will not treat the un-acked (from their POV) message as part of the
consistent transcript - it is not in their interest to do so. So we don't need
to *guarantee* that our own explicit acks are fully-acked. This avoids the
Byzantine agreement problem.

We *could* provide a guarantee even in the case of explicit acks, though. In
the next chapter, we'll talk about heartbeats, which as hinted before, are
automatic explicit acks, that result in an infinite sequence of mutual acks.
But, this is unsuitable for some high-latency or asynchronous protocols, so we
allow for the option to omit heartbeats, and talk about these separately.

The resend/dedupe scheme outlined above is quite minimal in the messages it
chooses to resend. Of course we may do more active resending; however this may
open up the path to DoS attacks. More research can be done here; for now we
have a good basic scheme with a clear logical justification, so we'll continue
with other topics.

.. _Pond: https://pond.imperialviolet.org/tech.html

.. [#Nack] The user could avoid reading the messages, but we can't do anything
    about this, and it's dubious that this could be abused for benefit. In the
    *average* case, there is some understanding.

Transcript consistency
----------------------

Transcript consistency may be constructed simply out of our message consistency
primitives. When we want to part, we send a "intend-to-part" message zm then
wait for explicit acks to it. Once it is fully-acked, we reach consistency for
all of anc(zm), which we'll treat as the transcript. Since we won't have any
future chances to receive the acks, we should wait longer than usual for this.

If we receive implicit or explicit acks to any message other than zm, we must
ignore them or display/log them permanently with a warning, because we have no
chance to verify their consistency. The same goes for implicit acks to zm.

If we fail to reach full-ack for zm (i.e. time out waiting for it), then we
still have consistency for all the messages we had previously reached full-ack
for, which hopefully is almost equal to anc(zm). For all other messages, we
should display/log them permanently with a warning.

TODO: construct a similar mechanism for cases where someone else wants to force
us to part.

This ensures consistency for messages that we delivered locally, including ones
we sent. What about messages by others that we haven't delivered, including
ones we haven't even received? We'll talk about this in the next section.
