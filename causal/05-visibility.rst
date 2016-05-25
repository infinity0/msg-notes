Partial visibility
==================

Member-sets
-----------

- diff and apply

Operations on the causal order
------------------------------

Define "cut" in the graph, generalisation of "before" and "after".

Expand definition of "acked-by".

Expand algorithms for FIN/ACK.

Subjective vs objective views
-----------------------------

- Rejoin semantics

Merge algorithm
---------------

Under partial visibility, our merge algorithm from the previous section does
not work consistently. Here are some examples that indicate the problem.

(Taken literally, they ignore that by(u) must be a total order; but we can
regain this property by rewriting a instead as multiple members {a1, a2, a3,
...}, however many is necessary. But for our current purposes, it's easier to
follow if we shortcut this and just write "a" instead.)

In this example, a has full visibility of the history, and can execute
the merge as normal:

.. digraph:: merge_full_visibility

    rankdir=BT;
    node [style="filled"];
    label="visible to a, not b";

    O [label="a"];
    A [label="auv"];
    A1 [label="au"];
    A2 [label="av"];
    A -> O [label="+uv"];
    A1 -> A [label="-v"];
    A2 -> A [label="-u"];
    subgraph cluster_d {
      label="visible to a, b";
      B [label="ab"];
      C [label="abu",fillcolor="#6666ff"];
      D [label="abv",fillcolor="#6666ff"];
      X [label="ab",fillcolor="#66ff66"];
      C -> B [color="#666666"];
      D -> B [color="#666666"];
      X -> C [color="#666666"];
      X -> D [color="#666666"];
    }
    B -> O [label="+b"];
    C -> A1 [color="#666666"];
    D -> A2 [color="#666666"];

But if b executes the merge, they have an incomplete view of history, and get a
different result:

.. digraph:: merge_partial_visibility

    rankdir=BT;
    node [style="filled"];

    B0 [label="ab"];
    C0 [label="abu",fillcolor="#6666ff"];
    D0 [label="abv",fillcolor="#6666ff"];
    X0 [label="abuv",fillcolor="#66ff66"];
    C0 -> B0 [label="\"+u\""];
    D0 -> B0 [label="\"+v\""];
    X0 -> C0 [color="#666666"];
    X0 -> D0 [color="#666666"];

Furthermore, b cannot distinguish between this history, vs an alternative
history where a does actually get the same result as b:

.. digraph:: merge_full_visibility_alternative

    rankdir=BT;
    node [style="filled"];
    label="visible to a, not b";

    O [label="a"];
    A1 [label="au"];
    A2 [label="av"];
    A1 -> O [label="+u"];
    A2 -> O [label="+v"];
    subgraph cluster_d {
      label="visible to a, b";
      B [label="ab"];
      C [label="abu",fillcolor="#6666ff"];
      D [label="abv",fillcolor="#6666ff"];
      X [label="abuv",fillcolor="#66ff66"];
      C -> B [color="#666666"];
      D -> B [color="#666666"];
      X -> C [color="#666666"];
      X -> D [color="#666666"];
    }
    B -> O [label="+b"];
    C -> A1 [color="#666666"];
    D -> A2 [color="#666666"];

Notice how the subgraph "visible to a, b" is identical in both cases, including
even its edges to nodes outside the subgraph. We conclude that there is no way
for "b" to properly execute the merge algorithm.
