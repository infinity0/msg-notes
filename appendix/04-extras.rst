============
Extra topics
============

.. _consistency-without-reliability:

Consistency without reliability
===============================

Against our main strong ordering proposal, here we'll look at how to construct
a consistency system that doesn't assume a reliable transport. (Spoiler: we
don't have a good solution to this problem, but we'll talk about the issues.)
Fundamentally, transcript consistency cannot be guaranteed since we accept that
members might miss any subset of messages. However, we can still try to achieve
consistency of individual messages - i.e. if Alice sends me a message that she
claims was also sent to X, then I should verify that X indeed received this.

Suppose Alice sends messages (1) and (2) to Bob. The transport is unreliable
and Bob receives (2), and we don't want to assume he will eventually receive
(1). At this moment, we can still indicate all necessary information to the UI:
on Alice's side, Bob hasn't acked her messages yet; on Bob's side, he has
received (2) which refers (optionally) to the missing parent (1).

Suppose then Bob wants to send message (3). Achieving consistency is based on
acking what we have seen so far, which means we need a way to *represent* this
information. By definition, our previous assumption of strong ordering, means
that we only ever need to represent "we have seen everything up to and
including X", where in this case X = {2}.

By contrast, without a reliability assumption, we would need to represent
arbitrary subsets of the transcript, e.g. in this case "seen (2) but not (1)".
As a strawman proposal, this is possible simply by explicitly enumerating the
individual messages we have seen, but this means that every member has to
explicitly transmit every message ID, which is not very efficient. This could
be an option for high-bandwidth unreliable transports however.

One strategy has been to work towards a system that can detect inconsistency,
yet may be unable to recover from specific inconsistent scenarios (since this
is unnecessary if we are OK with a unreliable transport), in the hope that such
a system would be simpler than strong ordering and not require buffering.

.. TODO(xl): explore the below further, needs more research

.. Can we simply display messages immediately when we receive them, yet keep our
   previously proposed semantics (or something similar to them) for parent pointers?

.. When the user then sends a message m, any parents p of m are only valid to one
   level, because we didn't wait for previous ancestors before showing p to the
   user. That is, a message m with pre(m) only means that the sender saw pre(m)
   and not all of anc(m).

.. Suppose Alice sends (1) (2) then (3), and Bob receives (1) and (3). Then, his
   next message (4) might point to (1, 3) because he doesn't know that (3) is a
   child of (1).

Nothing concrete has thus far been proposed, but we believe the goal itself to
be naive. In unreliable transports, drops and out-of-order deliveries happen
fairly frequently by definition, so users will often get false positive
"inconsistency" warnings, training them to ignore such warnings and reducing
their effectiveness.

We further suspect that such strategies would be able to unable to switch an
inconsistent to a consistent state, when out-of-order delivery occurs but all
messages are eventually received (i.e. consistency recovers when the transport
recovers), whereas buffering would be able to achieve this.

Timestamps
==========

We outline some techniques that try to determine clock skew, and detect
severely incorrect claims - but acceptance of a timestamp should not be treated
with substantial authenticity by any higher-lever application.

Semantics
---------

Define:

Declared vs local timestamps

Send vs recv timestamps
- recv might be either "delivered" vs "inferred-seen", elaborate
- the latter is also a total ordering like "delivery order" but would cause
  UI re-drawing.
- the latter probably more suitable for inferring remote timestamps

Some properties of these, assuming arbitrary delays in the network. Assumes
"global time" exists, i.e. not relativistic.

Minimum bound
-------------

Minimum local send timestamp (mlst)

Recursive definition of mlst, based on dst from others, and lst (== dst) of self.

Consequences of false declarations:

- dst too low -> as if network had much higher latency.
  mlst estimates might too generous, not such a big deal. it might even be
  unaffected since it's calculated as max() of information from multiple people.

- dst too high
  - if no-one notices -> no problem, as if network had much lower latency -
    actually a good thing!
  - if someone notices, send a "bad timestamp" message, emit a UI warning "mlst
    appears to be from the future; mlst calculations of later messages may also
    be affected by this"
    - diagram of example situation
    - how to handle (might be innocent)? getting quite complex, leave for
      future research

Could be integrated into freshness checks, but complexity probably not worth it.
