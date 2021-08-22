---
layout: post
title: Qick tip: Checking the BGP best path reason in IOS-XR 
slug: quick-ios-xr-tips-bgp-best-path-reason
---

Short tip, in case you want to quickly know the reason why an IP prefix has been selected in BGP, you can use the `best-compare` keyword when querying the prefix.

```Python
XR1#show bgp ipv4 unicast 8.8.8.0/24 bestpath-compare
Paths: (2 available, best #2)
  Advertised to peers (in unique update groups):
    10.1.2.2
  Path #1: Received by speaker 0
  64513 15169
      Origin incomplete, metric 0, localpref 100, valid, external, group-best
      Received Path ID 0, Local Path ID 0, version 0
      Origin-AS validity: not-found
      best of AS 64513
      Longer AS path than best path (path #2) << !Reason for not being best
  Path #2: Received by speaker 0
  64666
    10.1.3.3 from 10.1.3.3 (3.3.3.3)
      Origin-AS validity: not-found
      best of AS 64666, Overall best << !
```
As seen, the reason why the best path is path #2 is due to the fact that path #1 has a longer AS path.
