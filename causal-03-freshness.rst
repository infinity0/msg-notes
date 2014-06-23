======================
Freshness and presence
======================

Non-causal drops
================

As mentioned at the end of the previous section, we are so far unable to detect
non-causal drops - drops of messages that are not an ancestor of messages we
*have* seen and delivered. This does not affect transcript consistency, as
already argued. However, sometimes we might still want this as a security
property, especially in low-latency protocols.

One way we can do this is via fresh nacks - messages that we can be sure are
fresh (as of time t), that say "no messages sent by u since t". Our by(u) total
ordering encodes the "no messages since" meaning implicitly, so we just need to
prove freshness since time t.

A message m implies its sender was present at time of context(m)(self), if it
exists - we trust our own timestamps, and the partial order establishes the
"after" relationship. If context(m)(self) does not exist, then we'll have to
use something else written by ourselves, such as e.g. a key exchange message
that helped to derive the secrets that m is auth-encrypted by - we don't have a
mechanism to securely trust others' timestamps, and won't try to construct one
here. Note that we just store our timestamps locally, we don't need to send
them anywhere - and they wouldn't trust ours anyway.

The above achieves *end-to-end* security for freshness. By contrast, XMPP
presence is not end-to-end secure - it assumes a trusted server. Neither is
looking at "local receive time of a packet" - it assumes a trusted network.

Heartbeats
----------

We detect non-causal drops by periodically sending out heartbeats, and waiting
for these to be fully-acked (i.e. setting an ack-monitor on them); this proves
freshness as of the time of the heartbeat. To be efficient, we can omit a
heartbeat if we've already sent another message inside the previous period,
excluding explicit acks which don't need to be acked. We may refer to these
messages as "implicit heartbeats", akin to implicit acks.

If someone takes too long to ack a (implicit or explicit) heartbeat, we can
indicate this in the UI, for example by dulling the user avatar.

If heartbeats are regular, they may be distinguished from actual messages even
to an eavesdropper who can't read their contents. (The same may be said of
regular explicit acks and identical-ciphertext resends, though these probably
reveal less information.) In general, metadata security is a more complex
topic, and it is unclear whether these things are a real security problem to be
worth spending effort on, so we'll note the issue but won't pursue it here.

Presence and expiry
===================

Freshness is an indicator of presence, which we'll define roughly as "a good
chance a user will ack messages sent to them". We adopt end-to-end definitions,
so "presence" is considered from the local user's perspective, rather than the
internet - if a user's connection goes down, from their POV everyone else is
absent, rather than themselves. Typically, presence statuses must expire unless
explicitly refreshed, to be able to reliably indicate absence.

Our heartbeats are a rudimentary indicator of presence, one that is only partly
useful since it works within an already-established session. There are many
end-to-end secure presence mechanisms that are more general purpose, e.g. TODO.
Yet other mechanisms are simple but still effective - e.g. detecting that the
local network interfaces are disconnected, is a strong indicator of absence.

Locally, we may integrate our heartbeats with these other mechanisms, for more
accuracy and reliability. The exact logic will depend on the mechanism and what
interfaces it provides for integration, but it should be fairly intuitive and
straightforward. For example, the combined system should indicate presence when
a user acks a heartbeat *or* any external mechanism indicates presence, and any
of these should reset any expiry-based absence indicators.

Absence is semantically distinct from parting the session - absent users still
have the *right* to see messages, until they formally stop being a member of
the session. This distinction supports the ability to have a long-running
session where users go online and offline, but still want to see what was said
during their absence. This is common for some existing hosted chat systems, but
we can do this end-to-end too.

The feature of long-term absences must be considered against the requirements
of the greeting protocol. For example, some GKEs require all members to be
present in order to complete execution. Then, supporting long-term absences is
problematic, because this support will likely result in most of the session's
lifetime having partial membership presence. To avoid this, we recommended that
systems choose a greeting protocol that supports the forced-parting of absent
members *without their input*.

If a session is active whilst a user is absent, this will cause non-full-ack
warnings to be emitted. But this is expected because they are absent, so we can
account for it: if the users who haven't acked a message are all absent, the
severity of its non-full-ack warning should be reduced. Furthermore, the rate
of resends should be reduced too, since we believe they won't succeed anyway.
(However, if we have no other external presence mechanism, they should not be
completely stopped - otherwise all parts of a partition will stop sending, and
we'll never regain presence.) Later, when the absent user returns, the rate of
resends should be restored, or even reset to the maximum.

Timestamps
==========

So far we have avoided communicating any timestamps in the protocol, nor to
rely on specific timestamp values for others' messages. (Heartbeat timestamps
are only used locally as a lower-bound.)

However, specific values for timestamps are useful as a UI indication. How do
we do this?

Limitations
-----------

TODO Clock reliability.

- new device, time not set
- when travelling, some users change system clock instead of time zone
- change system clock for testing and forget about it

Verifiability.

If a member lies in the contents of a message, this is not something the
messaging protocol can detect. We treat timestamps as a similar sort data - no
other part of the system or the security models depends on them being even
vaguely correct.

Simplest option is to use local delivery time of a message. Additionally, each
message may contain the claimed sender local time, which may be displayed on
the side.

See appendix (TODO: link) for discussion of some more complex techniques, which
may result in better accuracy when users have skewed clocks. However, they are
quite complex and don't achieve very strong end-to-end security (as mentioned
above), which is why they are only in the appendix.
