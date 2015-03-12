===============
Causal ordering
===============

In general, messages are not self-contained but implicitly refer to previous
messages (their "context" or "causes"), and this is critical to their meaning.
A "context rebinding" attack presents a message with a different context, which
means recipients will interpret it differently from what the author intended.
We must guarantee not only a message's contents, but its context as well.

One nice property is that members should be able to send messages without
requiring approval from other parties, or equivalently, potentially having
messages rejected by others. Another nice property is to have a total (linear)
order for our messages that is globally consistent. However, this is impossible
if we want to preserve context and also retain the no-approval/no-rejection
premise. Proof: the basic distributed case is where two members each publish a
message with the same context O, call these a, b, without having seen the other
message first. Forcing these into a total order, WLOG say (O, a, b), would add
a to the apparent context of b, potentially causing it to be interpreted
incorrectly. Note that this holds regardless of how the total order is formed -
including e.g. by reflecting all messages via a server or leader - as long as
we don't require approvals or allow rejections.

If we abandon the no-approval premise, then we can achieve a context-preserving
total order, by using some mechanism (e.g. a consensus protocol) to approve one
message at a time in sequence. However, this has several downsides. It requires
interaction with other members of the session *for every single message*, which
is not feasible in an asynchronous scenario. Rejections must be handled
manually - automatic retries are effectively a context rebinding attack upon
yourself [#Narej] - but we consider this an unacceptable user experience.
Finally, the guarantees of consensus algorithms are much lower than the
computational security guarantees on context in our system.

So, we retain the no-approval premise and a causal order to represent the true
underlying history. We explore how to execute session mechanics and achieve
security properties using this structure. We ignore approaches that try to make
everyone agree on a total order. However, we do explore linearisations of the
causal order, but only for user interface purposes and not to reason about the
relative ordering of messages.

As will be discussed, some of the sub-strategies may be viewed as performance
penalties, such as temporarily preventing certain messages from being shown;
applications that can accept weaker ordering guarantees, such as streaming
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
policy-enforced membership operations.

.. [#Narej] Say we try to send a message m with context O and it is rejected.
    This means some other message m' was accepted also with context O. If we
    automatically retry m now, it will have m' in its context, even though the
    author of m originally wrote it without such a context.

Subtopics:

.. toctree::
   :glob:

   *
