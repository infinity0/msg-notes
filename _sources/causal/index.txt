===============
Causal ordering
===============

In this chapter we build a solid semantic data model for our group session, so
that we can build precise security properties on top of it later. This does
*not* mean we ignore security; on the contrary, we solve distributed systems
topics specifically *given* our security constraints, based on the principle
that *if you don't define something, an attacker will define it for you*.

We begin by describing some properties that we would like our system to have.

In general, messages are not self-contained but implicitly refer to previous
messages (their `context` or `causes`), and this is critical to their meaning.
A `context rebinding` attack (including `replay attacks`) presents a message to
the victim recipient under a different context, in the hope that the latter
interprets it differently from what its author intended. We'd like to guarantee
not only a message's contents, but its context as well.

We'd also like members to be able to send messages without requiring approval
from others (or the system as a whole, or any other entity) - or equivalently,
potentially having messages rejected by them. Another nice property is to have
a total (linear) order for our messages, that is globally-consistent.

Our first obstacle is that we cannot achieve all three properties. [#ordp]_ So
which one shall we abandon? If we abandon the no-approval property, then we can
achieve the others, by using some mechanism (e.g. a consensus protocol) to
approve one message at a time in sequence. However, this has several downsides.
It requires interaction with other members of the session *for every single
message*, which is not feasible in an asynchronous scenario. Rejections must be
handled manually [#arej]_ by the user, but we consider this an unacceptable
UX. Finally, the guarantees of consensus algorithms are much lower than the
computational security guarantees we can achieve when preserving context.

We consider it unacceptable to abandon the context-preserving property, since
this is a security issue. To abandon the globally-consistent total order has
much less severe consequences. In the messaging layer, we can execute all our
session mechanics, and reason about all our security properties, in terms of
the context-preserving partial (causal) order of messages. Then, in the UI, we
can recover most of the UX advantage that total orders have over partial
orders, by showing a subjective (non-global) linearisation of the "true" order.

The rest of this chapter will build on top of these choices, and further
explore their consequences. [#cseq]_ We'll also explore more complex mechanics:
in the previous paragraphs, we talked about messages, but we'll also explore
the semantics of membership operations and privacy across subsessions as well.

(Side topic: some key-rotation ratchets implicitly preserve context, since the
dependencies of which previous keys are used to protect each message matches
the actual causal order of messages. Our explicit formal treatment here works
outside of any ratchet, but we'll return to the relationships between ratchet
message dependency and causal orders in :doc:`another chapter <../ratchet>`.)

.. toctree::
   :glob:

   *

.. [#ordp] The basic distributed case is where two members each publish a
    message with the same context O, call these a, b, without having seen the
    other message first. Forcing these into a total order, WLOG say [O, a, b],
    would add a to the apparent context of b, potentially causing it to be
    interpreted incorrectly. ∎ Note that this holds regardless of how the total
    order is formed - including e.g. by reflecting all messages via a server or
    leader - as long as we don't require approvals or allow rejections.

.. [#arej] Automatic retries are effectively a context rebinding attack upon
    yourself: suppose we try to send a message m with context O and it is
    rejected. This means some other message m' was accepted also with context
    O. If we automatically retry m now, it will have m' in its context, even
    though the author of m originally wrote it without such a context. ∎

.. [#cseq] Certain consequences we derive may not be appropriate for some
    applications. For example, we temporarily prevent some messages from being
    displayed in the UI, in some cases. Applications that can accept weaker
    ordering guarantees, such as streaming non-sensitive video, may prefer to
    avoid this and choose a different system. However, one should be careful
    when making this security tradeoff: too often people claim "tradeoff" but
    in reality they only considered the usability or performance gain, without
    quantitatively understanding the security loss.
