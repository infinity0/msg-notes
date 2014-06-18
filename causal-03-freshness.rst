==========================
Freshness and availability
==========================

Detecting non-causal drops
--------------------------

Heartbeats
----------

Availability and expiry
-----------------------

Drop vs part - dropped users still have right to see messages, until
manually kicked.

When dropper reconnects, messages ought to be re-sent to them
pro-actively.

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
