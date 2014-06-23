===============
Causal ordering
===============

In general, messages are not self-contained but implicitly refer to previous
messages (their "context" or "causes"), and this is critical to their meaning.
A "context rebinding" attack presents a message with a different context, which
means recipients will interpret it differently from what the author intended.
We must guarantee not only a message's contents, but its context as well.

We work under the premise that members should be able to send messages without
requiring approval from other parties. The basic distributed case is where two
members each publish a message with the same context o, call them a, b, without
having seen the other message first. Interpreting these messages within a total
(linear) order, say [a, b], would change the context of b, adding a into it. So
a total order cannot preserve context, under the no-approval premise.

If we abandon the no-approval premise, and use a consensus algorithm to approve
messages, then we achieve a total order. However, this requires interaction
with other members of the session. We believe the cost of this is fatal to a
normal user experience, and destroys any ability of the system to work in an
asynchronous scenario. It increases the complexity of implementation, and the
guarantees it provides are still much lower than the computationally-secure
guarantees on context in our system. We will not discuss these further, but
other projects are welcome to take this approach.

From these initial considerations, we choose a causal order for representing
the relationships between messages, with a secondary total order with weaker
guarantees used only for user interface purposes. We explore how to execute
session mechanics and achieve security properties using this structure. As will
be discussed, some of these strategies may be viewed as performance penalties,
such as temporarily preventing certain messages from being shown; applications
that can accept weaker ordering guarantees, such as streaming non-sensitive
video, may prefer to avoid this and choose a different scheme.

Some key-rotation ratchets implicitly preserve context, since the dependencies
of which previous keys are used to protect each message matches the actual
causal order of messages. Our explicit formal treatment here works outside of
any ratchet, but we'll return to the relationships between ratchet message
dependency and causal orders in another chapter.

We also explore more complex mechanics. For concurrent membership operations,
we derive a merge algorithm over causal orders that satisfies intuitive notions
of quasi-global consistency. Our analysis includes conditions on how to encode
a global state to be compatible with this algorithm, which may be useful for
policy-enforced membership operations.

Subtopics:

.. toctree::
   :glob:

   causal-*
