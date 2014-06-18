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

Drop vs part - dropped users still have right to see messages, until
manually kicked. TODO: continue to ack-monitor or not? What happens if user
becomes present again?

May also integrate with external e2es presence mechanism.

When user absent, could throttle resends. Note however: if no external presence
mechanism, this should not go down to 0 - otherwise both parts of a partition
will stop sending, and we'll never regain presence!

When a dropper returns, messages ought to be resent to them more actively.
For example, if an ack-monitor is doing an increasing-interval strategy, we
could reset the interval to the minimum.

Timestamps
==========

So far we have avoided communicating any timestamps within the protocol, nor
to rely on specific timestamp values for others' messages. (Heartbeat timestamps
are only used locally as a lower-bound.)

However, specific values for timestamps are useful as a UI indication. How do
we do this?

Limitations
-----------

Clock reliability.
- new device, time not set
- when travelling, some users change system clock instead of time zone
- change system clock for testing and forget about it

Verifiability.

If a member lies in the contents of a message, this is not something the
messaging protocol can detect. We treat timestamps as a similar sort data - no
other part of the system or the security models depends on them being even
vaguely correct.

Simplest option then is to trust user's local time, as a UNIX timestamp.

See appendix (TODO: link) for discussion of some more complex techniques, which
may result in better accuracy when users have skewed clocks. However, they are
quite complex and don't achieve very strong end-to-end security (as mentioned
above), which is why they are only in the appendix.
