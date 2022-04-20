---
layout: post
title: "Tip: Graphing the SPT with IS-IS in IOS-XR" 
slug: isis-graph
---

Short and hopefully useful tip. If you want to graph the IS-IS topology, IOS-XR has a new command where this can be easily done. Below:

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

As seen, this gives us a result where the node name as well as its corresponding router ID is used to graph the topology. You can use a graphing online page such as https://edotor.net/ in order to graph it. For this particular topology, the result is seen below:

<img src="/assets/images/isis-graph.png" alt="">

Hope it helps in troubleshooting.

Elvin
