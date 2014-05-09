===============
Causal ordering
===============

Typical models of secure communication sessions only deal with two parties.
In this scenario, the session exists between single startup and shutdown
phases. From the point of view of each agent, there is only one *other*
partner to the session, and all other parties are untrusted outsiders.

When we extend communication to multiple parties, we bring up qualitatively
new issues. There are more mechanics to the session - members may join and
part within the lifetime of the session, outside of the initial startup or
final shutdown. There are multiple *other* partners, each of which may try
to act against each other. This demands additional security requirements,
such as ensuring that everyone sees the same consistent view of the session,
and that members can only read a partial view of the entire session, namely
between the time(s) they join and part. [#N1]_

Here, we present a scheme for dealing with these issues - namely a causal
ordering of messages, encoded by unforgeable immutable pointers. [#N2]_ On
top of the above, it also offers protection against network-level attacks
(reorder, replay, drop of packets) as well as simplifying the problem of
freshness. (These benefit two-party sessions too, but for whatever reason
aren't treated in much detail in existing protocols.) We also discuss
potential human interfaces of our data structure, including how to display
error conditions and the options for users to respond to these.

We believe this scheme allows extensions towards more complex mechanics. For
example, to support membership operations, we develop (we argue) a unique
algorithm for merging concurrent changes, that satisfies intuitive notions
of quasi-global consistency. We derive conditions on how to encode a global
state to be compatible with this algorithm, which may be useful for
policy-enforced membership operations. Later, we also discuss how the DAG
structure may be used as a guide to develop key-rotation ratchets based on
more complex key-agreement protocols, instead of 2-party DH that current
ratchets are based on.

.. [#N1] Some group communication applications do not have the latter
    requirement, but we think it is consistent with intuition for a
    "*private*" chat.

.. [#N2] These are implemented using cryptographic hashes, but the mentioned
    properties are the important interfaces to the overall scheme.

Subtopics:

.. toctree::
   :glob:

   causal-*
