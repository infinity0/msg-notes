===============
Causal ordering
===============

We'll begin by describing a few properties that we would like our messaging
system to have. In general, messages are not self-contained but implicitly
refer to previous messages (their "context" or "causes"), and this is critical
to their meaning. A "context rebinding" attack (which includes replay attacks)
presents a message to the victim recipient under a different context, in the
hope that the latter interprets it differently from what its author intended.
We'd like to guarantee not only a message's contents, but its context as well.

We'd also like members to be able to send messages without requiring approval
from others (or the system as a whole, or any other entity) - or equivalently,
potentially having messages rejected by them. Another nice property is to have
a total (linear) order for our messages, that is globally-consistent.

Our first obstacle is that we cannot achieve all three properties. [#Nordp]_ So
which one shall we abandon? If we abandon the no-approval property, then we can
achieve the others, by using some mechanism (e.g. a consensus protocol) to
approve one message at a time in sequence. However, this has several downsides.
It requires interaction with other members of the session *for every single
message*, which is not feasible in an asynchronous scenario. Rejections must be
handled manually [#Narej]_ in the UI, but we consider this an unacceptable user
experience. Finally, the guarantees of consensus algorithms are much lower than
the computational security guarantees we can achieve when preserving context.

We consider it unacceptable to abandon the context-preserving property, since
this is a security issue. To abandon the globally-consistent total order has
much less severe consequences. In the messaging layer itself, all the session
mechanics may be executed, and all the security properties reasoned about, in
terms of the context-preserving partial (causal) order of messages. Then, in
the UI layer, we can recover most of the UX advantage that total orders have
over partial orders, by using a subjective (non-global) linearisation of the
"true" order. The rest of this chapter will explore these topics.

Certain strategies we discuss may not be appropriate for some applications. For
example, we temporarily prevent some messages from being displayed in the UI.
Applications that can accept weaker ordering guarantees, such as streaming
non-sensitive video, may prefer to avoid this and choose a different scheme.

Some key-rotation ratchets implicitly preserve context, since the dependencies
of which previous keys are used to protect each message matches the actual
causal order of messages. Our explicit formal treatment here works outside of
any ratchet, but we'll return to the relationships between ratchet message
dependency and causal orders in :doc:`another chapter <../ratchet>`.

We also explore more complex mechanics. For concurrent membership operations,
we derive a merge algorithm over causal orders that satisfies intuitive notions
of quasi-global consistency. Our analysis includes conditions on how to encode
a global state to be compatible with this algorithm, which may be useful for
policy-enforced membership operations. TODO(xl): reword this later

.. [#Nordp] The basic distributed case is where two members each publish a
    message with the same context O, call these a, b, without having seen the
    other message first. Forcing these into a total order, WLOG say (O, a, b),
    would add a to the apparent context of b, potentially causing it to be
    interpreted incorrectly. Note that this holds regardless of how the total
    order is formed - including e.g. by reflecting all messages via a server or
    leader - as long as we don't require approvals or allow rejections.

.. [#Narej] Automatic retries are effectively a context rebinding attack upon
    yourself. Say we try to send a message m with context O and it is rejected.
    This means some other message m' was accepted also with context O. If we
    automatically retry m now, it will have m' in its context, even though the
    author of m originally wrote it without such a context.

Subtopics:

.. toctree::
   :glob:

   *
