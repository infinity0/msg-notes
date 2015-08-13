===============================
Indicating security information
===============================

Here we describe, in abstract terms, concepts that must be communicated to the
user during secure and insecure situations, so that they may distinguish them
and be able to take appropriate action. We do not advocate specific concrete UX
features here (though we give a few suggestions as examples), but we *do*
intend for a designer to use these discussions to develop such proposals.

Integrity and reliability
=========================

Real parents of a message
-------------------------

As explained elsewhere, we don't use a consensus mechanism for approving
messages, and therefore the session transcript is necessarily a partial order
and not a total (linear) order - even if there is (e.g.) a central server
dictating a secondary linear order for display purposes.

This means that some messages may not be displayed directly below the one(s)
that they are directly replying to. Most of the time this isn't a problem, and
human users can naturally resolve any ambiguity. However, for cases where this
is not possible, or if the session is security-critical, then there should be a
secondary interface to indicate the real parents/ancestors of a message.

Some suggestions include:

- annotations to indicate real parents, for example::

    Alice: Who likes ice cream?
    Bob: Who likes SSLv3?
    Carol: Me! [2] # indicates replying to -2nd message relative to this one

- highlight (gray out) a message's (non-)ancestors on hover / long-press

Messages sent during a member change
------------------------------------

TODO(xl): generalise so these aren't specific to GKAs

It takes *some* time to run the GKA (group key agreement) to change the session
key to include/exclude a user. During this time, messages may be sent (by
anyone) to the older set of members. When the GKA succeeds, there might be a
race condition like this::

    Bob   sends: (1)               # final message of final round of GKA
    Carol sends: (2)               # message encrypted to old members
    Alice recvs: (1) from Bob      # Alices finishes GKA, and updates her view of the membership
    Alice recvs: (2) from Carol

In the above diagram, (1) and (2) are messages. Carol sends (2) before she has
received (1) from Bob. When Alices receives (1), her UI will say something like
"Dave has joined the room" and her next message will be encrypted to Dave - but
later she will receive (2) from Carol that was *not* encrypted to Dave.

The UI must be able to show that (2) was sent to the *old* members rather than
the new members. We know this information in the underlying system; the problem
is how to *indicate* it in a non-confusing way.

Own messages sent after starting a member change
------------------------------------------------

Some membership operations may take some time to complete. If we exclude a
member, and the greeting system does not immediately make this effective, then
the UI for sending messages must *clearly indicate* that messages we send are
still being encrypted to the old membership, that includes the member in the
process of being excluded.

(The issue also exists for joining, but is less serious since any message
intended for the newer group implicitly has permission to be seen by the older
group.)

Some suggestions include:

- Disabling the send button whilst the exclusion operation is in progress.
  This degrades the user experience, and should be avoided.

- Have explicit notices that a member was included / excluded - in addition to
  the user list, which is typically not in the primary area of the screen. When
  the local user excludes a member, there must already have been an inclusion
  notice for that member, (hopefully) conveying to the local user that a
  temporary lack of an exclusion notice means they are not yet excluded.

.. _hci-ordering:

Messages received out-of-order
------------------------------

We guarantee ordering - roughly, that messages are seen by recipients in the
order in which they were sent - by buffering messages received out-of-order.
Say Alice sends messages (1) and (2) to Bob. If the transport is unreliable and
Bob receives (2) first, then we won't display (2) until we receive (1), so
that we can display both in the correct order.

If we don't receive (1) after a reasonable time, we *might* try to communicate
this to the user - e.g. a notice like "(2) received out-of-order; waiting for
older (1)". Importantly, we must not display the *contents* of these messages
in waiting, to avoid breaking our security assumptions. [#buf]_ We could also
offer some advice on why this might be happening, but in a way that avoids
giving a false impression on *the exact cause*: the symptom could either
indicate an unreliable transport, or an attack by the transport, or even
another member, but *we don't know which*.

On the other hand, the above may add unnecessary complexity to the UX, and
burden the user with information that is hard to take action on. So, we might
just simply ignore when messages are being held in the buffer - i.e. to not
reveal this to the user at all, and pretend that we haven't received (2), to
avoid frustrating the user. This may seem unnatural at first, but it is quite
common for transport schemes that provide ordering guarantees - e.g. in TCP, if
higher-sequence packets arrive earlier than lower-sequence ones, they are
buffered completely invisibly to higher-layer applications and to the user.

As a failsafe to this simple option, we can trigger a warning if held messages
remain in the buffer for too long. Optionally, we could automatically end the
session and then display the contents of the buffered out-of-order messages -
this is safe because there are no further messages to be sent, so there is no
longer any chance of breaking the aforementioned security assumptions.

.. [#buf] Whenever we send a message, we also implicitly acknowledge all
    previous messages, in an efficient way. If we send a message after showing
    some messages but not their parents, then this acknowledgement is a lie
    that propogates and corrupts other members' views of the session and its
    consistency. See :ref:`consistency-without-reliability` for a more detailed
    exploration of this.

Messages not yet acknowledged
-----------------------------

After a message is displayed (authored by either us or others), we wait for
everyone else to acknowledge it. During this time, the message is "not
fully-acked", and we also know exactly who has not yet acknowledged it.
Ideally, this set of people will eventually become empty, at which point we
know that everyone has seen it ("fully-acked"). The question is how to display
the "not fully-acked" status during the interim period. For example:

- We could gray out the message until it is fully-acked.

  - A variant of this is to gray it only after a grace timeout period, if it is
    not already fully-acked by that time. The reasoning behind this is that
    *every* message will take *some* time for everyone to ack it, but graying
    *every* message immediately when displayed might be annoying or weird.

- We could add extra indicators to a message when it is fully-acked.

For comparison: At the time of writing, Facebook chat with 4 people looks
something like this::

    Alice : (1)
    Dave  : (2)
    Carol : (3)
    [seen by Bob]

This means Bob has seen {1,2,3} - this is the equivalent of "acked" in our
system. (Ours is more secure, but the *intended* semantics are the same.) With
our strong ordering guarantees mentioned above, we can additionally deduce that
Carol has seen {1,2,3}, and Dave has seen {1,2} - but what has Alice seen? She
might have seen {1,2} or only {1}; it would be good to let the user know
exactly which. Yes, this extra information is likely unnecessary in "most
typical cases", but it is still good to make it accessible in a secondary
interface, in case the user has critical needs for a particular case.

As with drop detection, if we reach a good state (here, consistency) *after*
the timeout, we should cancel the warning or at least downgrade its severity.
Also, here the warning is associated with a message that is *already displayed*
to the user. Therefore, it is important that the warning is shown not as a
point-event for the user to dismiss and forget, but as a persistent state
shown together with the message, in the same space and time. TODO(xl): reword

Users not responding to heartbeats
----------------------------------

We optionally send heartbeats to check that other people are alive. The purpose
of this is to detect that "no messages received from Dave" really means "Dave
didn't send any messages", as opposed to "the attacker dropped all of Dave's
messages".

There could be some indication of this in the UI - for example, graying out a
username when we don't see their heartbeat responses, etc. Extra information
could be provided via a secondary interface, like "user not responsive since 19
seconds ago".

Confidentiality and authenticity
================================

TODO

Other considerations
====================

Secondary interface
-------------------

In most typical cases, the user may not care too much about the nuances of some
security properties (e.g. group integrity and reliability), and therefore
shouldn't be burdened by the extra security information that our system offers.
However, in some cases, it may be a strong user concern, so we should provide a
way for the user to access this information; it would be irresponsible not to.
The existence of these features are also a good way of distinguishing our
secure application from competing insecure applications; see next section.

By *secondary interface*, we mean UI features that exist away from the main
view, so as to not overload the user with too much information. This is meant
to achieve a smooth default UX for most users, but also make detailed security
information accessible for users with high security needs. One approach is to
provide extra information in pop-ups that are only activated by long-press or
double-click on a related component in the primary interface; designers may
evaluate a wide range of other possibilities too.

High-level summary indicators
-----------------------------

The properties above are fine-grained information about error conditions of
specific messages or users. However, the desired state is for there to be no
errors, and hopefully this should hold most of the time (i.e. with a
non-malicious and semi-reliable transport). Rather than scanning our eye
through an entire list of messages or users to check that all of them are OK,
it would be more efficient to have a small number of "summary" indicators.
Users can then tell at-a-glance if everything is OK - if so, there is no more
work to do; if not, *then* they can check a more time-consuming secondary
interface for specific information about errors.

For example, there could be one indicator for each of the properties mentioned
above, that is quantified over *all* messages/users:

- no out-of-order messages being queued for display = "not missing any older messages"
- seen a heartbeat for all users recently = "not missing any newer messages"
- all messages acked by everyone else = "everyone else has seen what we've seen"

The first two could even be combined into the same indicator, to communicate
"not missing any messages". However, it is probably good to separate the third
item, since it takes longer than the other two to reach assurance about.

"Indicator" is used loosely; of course it is OK and maybe preferable if they
are invisible for good states, and only show themselves when something is
actually wrong.

Relationships with existing and insecure notions
------------------------------------------------

Many messaging applications already have notions analogue to the ones described
above, but these are not end-to-end secure. For example, with XMPP presence,
the server tells the client when a contact is online/offline, or when they
join/part a channel. Other applications have an idea of delivery receipts, but
these are authenticated by the server rather than by the actual recipient.

Such information should be regarded as untrusted. We implement *more secure*
(i.e. end-to-end) forms of these notions, so the UI should emphasise the
properties from our system, and de-emphasise the less secure ones.

In some cases, it could be good to *combine* the information from both sources.
For example, if we haven't received an (authenticated) acknowledgement, we are
*unsure* if a message has been delivered or not. But, if the local internet is
offline, or if the server is unreachable, then we *know* that a message has
*not* been delivered. These differences in meaning could be communicated via
different states (e.g. colours) of the same UI indicator.

In other cases, the untrusted source may be *replaced completely* by our more
secure notions. For example, in the case of freshness, it is trivial for the
server to report false presences, so our system (authenticated heartbeats)
should be used where possible. For the case of joining/parting a room, this is
probably more appropriate too - i.e. hiding XMPP join/part events, and only
indicate GKA events in the UI.
