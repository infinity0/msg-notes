Partial visibility
==================

Member-sets
-----------

Diff and apply
``````````````

Subjective vs objective views
-----------------------------

Merge restrictions on member-set operations
-------------------------------------------

Invite vs rejoin
----------------

Merge algorithm
---------------

Under partial visibility, our merge algorithm from the previous section does not work consistently:

In the following example, b has full visibility of the history, and can execute the merge as normal:

.. digraph:: merge_full_visibility

    rankdir=BT;
    node [style="filled"];

    O [label="b"];
    A [label="abc"];
    A1 [label="ab"];
    A2 [label="bc"];
    B [label="bd"];
    C [label="abd",fillcolor="#6666ff"];
    D [label="bcd",fillcolor="#6666ff"];
    X [label="bd",fillcolor="#66ff66"];
    A -> O [label="+ac"];
    B -> O [label="+d"];
    A1 -> A [label="-c"];
    A2 -> A [label="-a"];
    C -> B [color="#666666"];
    D -> B [color="#666666"];
    C -> A1 [color="#666666"];
    D -> A2 [color="#666666"];
    X -> C [color="#666666"];
    X -> D [color="#666666"];

But when d executes the merge, he has an incomplete view of history:

.. digraph:: merge_partial_visibility

    rankdir=BT;
    node [style="filled"];

    B0 [label="bd"];
    C0 [label="abd",fillcolor="#6666ff"];
    D0 [label="bcd",fillcolor="#6666ff"];
    X0 [label="abcd",fillcolor="#66ff66"];
    C0 -> B0 [label="+a"];
    D0 -> B0 [label="+c"];
    X0 -> C0 [color="#666666"];
    X0 -> D0 [color="#666666"];
