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
members can only read a partial view of the entire session, namely between the
time(s) they join and part. [#N1]_

In this document, we discuss these core theoretical issues, and outline some
approachs for dealing with them. We assume a relatively-efficient broadcast
operation, that costs near-constant time with the number of participants.

First, we present a scheme for representing the distributed nature of group
communication - namely a causal order of messages, encoded by unforgeable
immutable pointers. [#N2]_ This protects against network attacks (reorder,
replay, drop of messages) as well as simplifying the problem of freshness. It
protects against malicious insiders, enforcing session transcript consistency
between all members. We also discuss human interfaces for this, including how
to display error conditions and the options for users to respond to these.

Next, we deal with what we call "greeting protocols" - protocols that operate
on the membership set of the session. We look at the tradeoffs between
peer-to-peer pairwise keys, per-sender keys, vs group keys.

Third, we look at how the causal ordering may be used as a framework to develop
key-rotation ratchets based on more complex greeting protocols, as an extension
of 2-party DH that current ratchets are based on, by analysing the dependency
relationships between the messages of the underlying protocol.

.. [#N1] Some group communication applications do not require this, but we
    think it is consistent with intuition for a "*private*" chat.

.. [#N2] These are implemented using cryptographic hashes, but the mentioned
    properties are the important interfaces to the overall scheme.

.. toctree::
   :glob:
   :maxdepth: 2

   causal
   greet
   ratchet
   appendices

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`

