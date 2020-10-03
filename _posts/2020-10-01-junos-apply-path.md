---
layout: post
title: Understanding Junos $apply-path feature
slug: junos-apply-path.md
---

Junos $apply-path is a feature that allows for secure and simplified configuration parsing of IP addresses within the Junos software. How does it works? A matching condition is created under a particular hierarchy (protocols, interfaces, etc.), based on this junos is able to get the 'values' (IP addresses) to be expanded based on the current configuration.

Let's see an example of how we can use appy-path. In this network we have setup pair of router connected via BGP between r1 and r2, due to security reasons we need to apply a `firewall filter` to reject any connection attempts to BGP port 179 coming from sources other than our explicitly configured external peers, also for future growth, we need to update the $prefix-list every time a new peer is added to the BGP group.

#### Image 1 - my $bgp_peers topology
```python
r1<------------->r2
.1 [10.1.2.0/24] .2
```

The configuration to accomplish this is fairly simple, since we know the IP address of our peers, and we craft a *firewall filter* discarding TCP port 179 (BGP).

#### Example 1 - Allowing r1 BGP peer, rejecting everything else with no appy-path.
```python
/* r1 firewall filter configuration */

set policy-options prefix-list BGP-PEER 10.1.2.2/32
set firewall family inet filter EDGE-BGP-PROTECTION term ALLOW-PEER-ONLY from source-prefix-list BGP-PEER
set firewall family inet filter EDGE-BGP-PROTECTION term ALLOW-PEER-ONLY from protocol tcp
set firewall family inet filter EDGE-BGP-PROTECTION term ALLOW-PEER-ONLY from destination-port bgp
set firewall family inet filter EDGE-BGP-PROTECTION term ALLOW-PEER-ONLY then count PEER-ONLY
set firewall family inet filter EDGE-BGP-PROTECTION term ALLOW-PEER-ONLY then accept
set firewall family inet filter EDGE-BGP-PROTECTION term ELSE-REJECT then count ELSE-REJECT
set firewall family inet filter EDGE-BGP-PROTECTION term ELSE-REJECT then reject
```

As observed this is fairly straightforward to configure, now r1 will only accept connections to TCP port 179 coming from r2. Now let's say r1 needs to peer with 50 more routers within that BGP group, this clearly becomes cumbersome since we would need to add new IP addresses per BGP peer we configure to the $prefix-list. This is prone error-prone and NetEng might forget to add the new peer IP addresses under the $prefix-list leaving the network unprotected.

This where $apply-path feature comes into play: What if instead of manually updating our $prefix-list every time a new neighbor is added, we could somehow inherit the peer IP address from the configuration itself? Tthis is then passed to a $prefix-list which will contain the list of peer IP addresses. If we add or delete any BGP peers from the configuration the $apply-path would be updated without NetEng intervention.

#### Example 2 - Configuring junos $apply-path
```python
/* r1 $apply-path */

set policy-options prefix-list BGP-PEER apply-path "protocols bgp group <*> neighbor <*>"
```
To verify what is the $apply-path expansion, we need to use the 'display inheritance' option when doing the config verification.

#### Example 3 - Expanding $prefix-list with $apply-path
```python
[edit]
root@r1# show policy-options prefix-list BGP-PEER | display inheritance
##
## $apply-path was expanded to:
##     10.1.2.2/32;
##
$apply-path "protocols bgp group <*> neighbor <*>";
```
As we add new BGP peers to r1's BGP group, the $apply-path would be automatically expanded with the new IP addresses from the neighbors.
#### Example 4 - Peer IP dynamically added to the $prefix-list
```python
[edit]
root@r1# show policy-options prefix-list BGP-PEER | display inheritance
##
## $apply-path was expanded to:
##     10.1.2.2/32;
##     10.1.2.3/32; << ! New prefix added automatically to the $prefix-list.
##
apply-path "protocols bgp group <*> neighbor <*>";
```

Finally as an additional note, beyond BGP, we can use $apply-path under NTP, RADIUS, TACACS, or any other arbitrary hierarchy ($system, $interfaces, $protocols) for the purpose of automatic address expansion within the junos configuration.
