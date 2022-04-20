---
layout: post
title: "Tip: Graphing the SPT with IS-IS in IOS-XR" 
slug: isis-graph
---

If you were troubleshooting an IS-IS issue, it is often useful to know the topology, all you need is in the link-state database (LSDB), as it posses the graph of the topology. Doing the mapping manually is not as fast as we wish, but thankfully IOS-XR has a new feature where we can ask for the topology graph. It can be useful while troubleshooting.

```perl
RP/0/RP0/CPU0:XR1#show isis database graph 
/*
 * Network topology in DOT format. For information on using this to
 * generate graphical representations see http://www.graphviz.org
 */
graph "level-2" {
  graph [rankdir=LR];
  node [fontsize=9];
  edge [fontsize=6];
  "XR1" [label="\N\n1.0.0.1"];
  "XR1" -- "XR2";
  "XR2" [label="\N\n1.0.0.2"];
  "XR2" -- "XR3";
  "XR3" [label="\N\n1.0.0.3"];
}
```

As seen, this gives us the result as a graph with the name of the node and the router ID. 

<img src="/assets/images/Graph.png" alt="">

Hope it helps in troubleshooting.

Elvin
