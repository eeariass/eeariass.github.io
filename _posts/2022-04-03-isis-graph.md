---
layout: post
title: "Tip: Graphing the SPT with IS-IS in IOS-XR" 
slug: isis-graph
---

If you were troubleshooting routing issue, it is often useful to know the topology, right? :)

For link-state protocols, all you need to graph the topology is inside the link-state database (LSDB), as it posses the view of the network. If your IGP happens to be IS-IS, IOS-XR has a new command where you can ask for the graph of the topology and it provides the resultant view, where you can use an online tool to visualize it.

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

This parse the LSDB and it gives us the graph of the topology, the result can be seen [here](https://is.gd/vcu9J5), as observed, the Node + Router ID are being used for this. 

<img src="/assets/images/Graph.png" alt="">

Hope it helps,

Elvin
