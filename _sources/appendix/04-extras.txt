============
Extra topics
============

.. _encoding-message-identifiers:

Defining message identifiers
============================

We need message identifiers (mIds) to be pre-image and 2nd-pre-image resistant:

- for a given id, nobody can forge a message with the same id
- for a given message, nobody can forge another message with the same id

Hash functions are designed to (hopefully) have these properties. So a simple
and easy-to-implement definition for an mId is to simply hash the ciphertext.
There are some potential issues with this approach:

- It assumes there is exactly one ciphertext per message that is sent to and
  verified by every recipient. However, some cryptographic schemes may require
  that different recipients verify *different* ciphertext e.g. "peer-to-peer"
  or "pairwise" schemes where each pair within the group shares a MAC key.

- Anyone who sees this blob (including network attackers) can calculate the
  message id. Nothing we have mentioned in this series of articles *requires*
  mIds to be secret; but it may be that an application that is built on top of
  our ideas would find it convenient to have this security property.

Therefore, we also look at more general and flexible ways of defining a mId.
These are more complex to implement, but have less caveats when deployed.

To start with, even if each message has multiple ciphertexts, by definition it
must have a single "plain content" that is identical across recipients. This
includes data *and* metadata such as parent mIds, author identifiers (though
this could be implicit in the authentication mechanism) and recipient
identifiers (though this could be implicit in the encryption mechanism).

We should not hash this plain content directly - it may have low entropy so
that potential hashes can be pre-calculated by an attacker. [#Nhash]_ What we
do instead is to hash a secret together with this plain content. Since we are
encrypting the content, the encryption key would be a suitable secret:

  mId(m) := H(k[m] || content)

If our encryption scheme does not naturally have a single secret key for each
message (e.g. in a "peer-to-peer" scheme, we encrypt separately to each
recipient), we can tweak it so: instead of calculating GroupEnc(recipients,
payload) directly, we generate a random k[m] and calculate GroupEnc(recipients,
k[m]) and Enc(k[m], payload), where Enc is a normal single-recipient encryption
scheme. The k[m] may then be used to calculate mId as above. This tweak should
be possible regardless of the implementation or state of GroupEnc.

.. [#Nhash] As previously, this is not a problem with anything we have directly
  mentioned, but it places unnecessary caveats on anyone wishing to build on
  top of it. For example, we may wish to be free to let new joiners see old
  hashes, but without letting them see the contents of older messages.

.. _author-specific-message-identifiers:

Author-specific identifiers
---------------------------

As mentioned before, seeing a message is not the only way that a member might
come to know its mId. Other members might reveal it, by referring to it in the
"parent mIds" part of their own messages. So, knowing an mId does not *prove*
that they saw this message. As also explained before, we do not think that this
is a security problem.

However, it is *possible* to devise a scheme to "declare parent messages" that
does not enable others to fake their own declarations. The idea is simple -
that parent mId declarations should only be valid for the member giving the
declaration. A simple way to accomplish this is to define:

  umId(m, u) := H(k[m] || content || u)

If member u wants to declare that its next message has parent m, it declares
umId(m, u) instead of mId(m). One can calculate umId(m, u) if one has seen the
message, and can verify others' values; but one cannot calculate this if one
hasn't seen the message, nor re-use others' values in a fake declaration.

However, bear in mind that we haven't had this "security property" nor "threat"
(of fake parent declarations) in mind when designing other aspects of the
system, so the above scheme may not be sufficient (but it is necessary) to
protect against this "threat" in the overall system.

For example, we define acks to be transitive - referring to a message implies
that you've also seen its ancestors, without needing to refer to them directly.
This bypasses much of the "protection" of the above scheme. We could fix this,
by e.g. having a member-specific "all messages acked" hash that attests (in a
non-transferrable way, like umId) to all messages that we expect to be acking
with this current message, that can be verified by others. But this is getting
very complex quite quickly, for questionable gain, so we'll stop here and leave
further progress along these lines as an exercise to the reader.

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
