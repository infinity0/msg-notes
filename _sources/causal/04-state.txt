Pseudo-mutable state
====================

Union of events vs pseudo-mutable shared state
----------------------------------------------

Global state
------------

3-way merge
-----------

Possible state types
````````````````````

.. _merge-algorithm:

General merge algorithm for DAGs
--------------------------------

Commutativity and associativity
```````````````````````````````

History matters
```````````````

Merging the same set of states {{ab}, {bc}} in both cases, but with different history:

.. digraph:: merge_history

    rankdir=BT;
    node [style="filled"];

    x1 [fillcolor="#66ff66",label="b"];
    z1 [label="abc"];
    l1 [fillcolor="#6666ff",label="ab"];
    r1 [fillcolor="#6666ff",label="bc"];
    l1 -> z1 [label="-c"];
    r1 -> z1 [label="-a"];
    x1 -> l1 [color="#666666"];
    x1 -> r1 [color="#666666"];

    x0 [fillcolor="#66ff66",label="abc"];
    z0 [label="b"];
    l0 [fillcolor="#6666ff",label="ab"];
    r0 [fillcolor="#6666ff",label="bc"];
    l0 -> z0 [label="+a"];
    r0 -> z0 [label="+c"];
    x0 -> l0 [color="#666666"];
    x0 -> r0 [color="#666666"];

Node labels indicate the state (an unordered set) at the node. Blue nodes are the input (nodes to be merged), green is the output (merged state), gray arrows indicate merge-parents (i.e. nodes with >1 parent).
