.. msg-notes documentation master file, created by
   sphinx-quickstart on Thu May  8 13:11:30 2014.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Topics in secure messaging systems
==================================

Current models of secure communication sessions only deal with two parties. In
this scenario, the session exists between single startup and shutdown phases.
From the point of view of each participant, there is only one *other* partner
to the session, and all other parties are untrusted outsiders.

When we extend communication to multiple parties, we discover qualitatively new
issues. There are more mechanics to the session - members may join and part
within the lifetime of the session, outside of the initial startup or final
shutdown. There are multiple *other* partners, each of which may try to act
against each other. This demands additional security requirements, such as
ensuring that everyone sees the same consistent view of the session, and that
members can read the session only between the time(s) that they join and part.
[#Npart]_ We also work under the premise that every member knows the session
membership, but again only for the periods that they actually participate in
the session. [#Npriv]_

In these documents, we discuss these theoretical issues and outline approaches
for dealing with them. We assume an efficient packet/cell/stanza broadcast
operation, that costs near-constant time with the number of participants. We
aim for a single but modular protocol system that can support both synchronous
and asynchronous operation (where not all members may be online simultaneously,
perhaps not even any two), whilst maintaining strong security properties across
all cases. [#Nasync]_

**Outline**. First, we deal with session group integrity. One can use existing
two-party protocols to multicast messages to several individuals, but nothing
guarantees the coherent existence of a *united group*. We discuss the topics
within this area, and the high-level design choices available. We then propose
a concrete scheme for solving these issues, namely a causal order of messages,
encoded by unforgeable immutable pointers. [#Nhash]_ This protects against
transport attacks (reorder, replay, drop of messages) and simplifies the
problem of freshness. It enforces session transcript consistency, protecting
against malicious senders. This section is primarily related to distributed
systems, secondarily to security and cryptography.

Next, we deal with session membership control - protocols that operate on who
is part of the session. (This includes session establishment, which is the only
case that needs to be considered in 2-party protocols.) We discuss the security
properties common in this area, as well as their practical importance. We look
at the tradeoffs between different schemes, in terms of description complexity,
runtime efficiency and security. This section is primarily related to security
and cryptography, and secondarily to distributed systems.

Third, we look at how the causal ordering may be used as a framework to develop
key-rotation ratchets based on more complex greeting protocols, as an extension
of 2-party DH that current ratchets are based on, by analysing the dependency
relationships between the messages of the underlying protocol.

Throughout, we discuss UI issues that arise from our proposals. We also present
a summary of generic group messaging UI issues, that all applications will need
to resolve.

.. [#Npart] Some group communication applications do not require this, but we
    think this more closely matches intuition for a "*private*" chat. Also, one
    may reverse this behaviour by building *on top of* hide-by-default, but one
    cannot reverse show-by-default - so it's more flexible to pick the former.
    (Similar to how one can reverse deniability but not non-repudiability.)

.. [#Npriv] An even more private setting would be to make non-contact members
    pseudonymous (e.g. using an ephemeral identity key), but this would involve
    more complexity so we'll overlook it for now. However, we believe that our
    proposals do not prevent this from being added in the future.

.. [#Nasync] It is probably true that asynchronous sessions *often* expect
    different security and timing/latency properties than synchronous sessions,
    but the former does not *necessarily* determine the latter. (TODO: give
    examples). So we aim for strong security properties *and* the flexibility
    of asynchronous operation. Further, all timing rules are proposed in terms
    of an abstract basic time unit, which can be chosen by the application.

.. [#Nhash] These are implemented using cryptographic hashes, but the mentioned
    properties are the important interfaces to the overall scheme.

.. toctree::
   :glob:
   :maxdepth: 2

   causal/index
   greet
   ratchet
   appendix/index

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`

