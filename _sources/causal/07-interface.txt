============================
User interface and workflows
============================

.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

Displaying a causal order
=========================

It's unclear the "best" way to draw a causal order - there are many layout
algorithms for general DAGs, that optimise for different aims. One approach we
believe is suited for group messaging applications, is to convert the causal
order into a total order (a linear sequence). These are simple to display, and
intuitive as a human interface for our scenario. To be clear, this artifical
order is only used for display; the core system, responsible for security
properties, works entirely and directly on the causal order.

Our first conclusion is that any linear order must not be the *only* interface
shown to a user. This is because *all* linear conversions are open to context
rebinding. The simplest case of a non-total causal order is G = (a |rightarrow|
o |leftarrow| b). WLOG suppose that our conversion turns G into (o, a, b) for
some user. Given only this information in a display, they cannot distinguish
(o, a) from (a, b) - yet a is not a real parent of b. This might lead to an
attack - if one user sees (o) and has a reasonable expectation that someone
will reply (b), then they may say (a) in the hope that (b) will be linearised
after it, rebinding its context.

To protect against this, implementions *must* distinguish messages whose pre(m)
does not equal the singleton set containing the preceding message in the total
order; and *should* provide a secondary interface that shows the underlying
causal order. This may be implemented as parent reference annotations, hover
behaviour that highlights the real parents, or some other UX innovation. For
example:

| Alice: innocent question?
| Chuck: incriminating question?
| Bob: innocent answer! [2]

The [2] means "the last message(s) the sender saw was the 2nd message above
this one". Messages without annotations are implicitly [1] - this has the nice
advantage of looking identical to the unadorned messages of a naive system that
does not guarantee causal order. In practise, only a minority of messages need
to be annotated as above; we believe this is acceptable for usability.

TODO
- could minor-warn/notice for messages that are "badly-ordered", or omit annotations for low numbers

There are several caveats when selecting a linear conversion, and we need to
consider the trade-offs carefully. Let us introduce a few properties that would
be nice to achieve:

- globally-consistent / canonical: the conversion outputs the same total order
  for every user, given the same causal order input

- append-only: it does not need to insert newly-delivered messages (from the
  causally-ordered input) into the middle of a previous totally-ordered output.
  In a real application the user only sees the last few messages in the
  sequence, and earlier ones are "beyond the scroll bar", so messages inserted
  there may never be seen.

- responsive: it accepts all possible causal orders and may work on them
  immediately, without waiting for any other input

Unfortunately, we cannot achieve all three at once. Suppose we can. WLOG
suppose our conversion turns G (from above) into (o, a, b) for all users. If
one user receives (o |leftarrow| b), and does not receive (a) for a long time,
a responsive conversion must output (o, b). But then when they finally receive
(a), they will need to insert this into the middle of the previous output.

Delivery order
--------------

We believe that global consistency is the least important out of the three
properties above, and therefore sacrificing it is the best option. It's
questionable that it gains us anything - false-but-apparent parents are not
semantic, so it's not important if they are different across users. As noted
above, we ought to have a secondary interface to indicate the real causal order
*anyway*, to guard against context rebinding.

A linear conversion follows quite naturally from topics already discussed: the
order in which we deliver messages (for any given user) is a total order, being
a topological sort of the causal order. It is practically free to implement -
when the engine delivers a new message, we immediately append it to the current
display, and record the index for later lookup. This conversion is responsive
and append-only, but not globally consistent. It is also nearly identical to
existing chat behaviours, save for the buffering of dangling messages. [#Nres]_
Given its simplicity and relatively good properties, we recommend this as the
default for implementors.

Another approach achieves global consistency, but sacrifices one of the other
properties: bounce all messages via a central server, which dictates the
ordering for everyone. This strategy is popular with many existing non-secure
messaging systems. If we embed the causal order, we can re-gain end-to-end
security, and protect against the server from violating causality or context.
Then, we must either sacrifice responsiveness by waiting until the server has
reflected our message back to us before displaying it, or sacrifice append-only
and be prepared to insert others' messages before messages that we sent and
displayed immediately.

In either case, we still need *another mechanism* to ensure that the server is
providing the same total ordering to everyone, the whole point of doing this in
the first place. TODO: outline a mechanism for this. Or, we could omit this
mechanism and fall back to "delivery order" as above; then "global consistency"
would be achieved on a best-effort basis, relying on the server being friendly.

.. [#Nres] One may argue that due to this buffering, we are sacrificing
    responsiveness; however there is no *additional* waiting beyond what is
    necessary in order to achieve causal consistency - as discussed before,
    sacrificing the latter results in less security and greater complexity.

Server-dictated order
---------------------

TODO

See mpCAT.

Others
======

How to display warnings etc.
