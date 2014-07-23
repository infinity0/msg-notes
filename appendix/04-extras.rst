============
Extra topics
============

Timestamps
==========

We outline some techniques that try to determine clock skew, and detect
severely incorrect claims - but acceptance of a timestamp should not be treated
with substantial authenticity by any higher-lever application.

Semantics
---------

Define:

Declared vs local timestamps

Send vs recv timestamps
- recv might be either "delivered" vs "inferred-seen", elaborate
- the latter is also a total ordering like "delivery order" but would cause
  UI re-drawing.
- the latter probably more suitable for inferring remote timestamps

Some properties of these, assuming arbitrary delays in the network. Assumes
"global time" exists, i.e. not relativistic.

Minimum bound
-------------

Minimum local send timestamp (mlst)

Recursive definition of mlst, based on dst from others, and lst (== dst) of self.

Consequences of false declarations:

- dst too low -> as if network had much higher latency.
  mlst estimates might too generous, not such a big deal. it might even be
  unaffected since it's calculated as max() of information from multiple people.

- dst too high
  - if no-one notices -> no problem, as if network had much lower latency -
    actually a good thing!
  - if someone notices, send a "bad timestamp" message, emit a UI warning "mlst
    appears to be from the future; mlst calculations of later messages may also
    be affected by this"
    - diagram of example situation
    - how to handle (might be innocent)? getting quite complex, leave for
      future research

Could be integrated into freshness checks, but complexity probably not worth it.
