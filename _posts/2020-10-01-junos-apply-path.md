---
layout: post
title: Understanding junos apply-path feature
slug: junos-apply-path.md
---

Junos apply-path is a feature that allows for secure and simplified configuration parsing of IP addresses within the Junos software. How does it works? A matching condition is created under a particular hierarchy (protocols, interfaces, etc.), based on this junos is able to get the 'values' (IP addresses) to be expanded based on the current configuration.

### Scenario 1: Enhancing BGP security with apply-path
In this network we have setup pair of router connected via BGP between r1 and r2, due to security reasons we need to apply a `firewall filter` to reject any connection attempts to BGP port 179 coming from sources other than our explicitly configured external peers.

#### Image 1 - my $bgp_peers topology
```
r1<|.1--------------.2|>r2
         10.1.2.0/24
```
The configuration to accomplish this is fairly simple, since we know the IP address of our peers, we know we have to create a `firewall filter` that allows port 179 from peer IP address.

#### Example 1 - Allowing r1 BGP peer, rejecting everything else with no appy-path.

```
# r1 configuration:

policy-options {
    prefix-list BGP-PEER {
        10.1.2.2/32;
    }
}
family inet {
    filter EDGE-BGP-PROTECTION {
        term ALLOW-PEER-ONLY {
            from {
                source-prefix-list {
                    BGP-PEER;
                }
                protocol tcp;
                destination-port bgp;
            }
            then {
                count PEER-ONLY;
                accept;
            }
        }
        term ELSE-REJECT {
            then {
                count ELSE-REJECT;
                reject;
            }
        }
    }
}
```

If a source IP address other than 10.1.2.2 attempts to connect to r1 via port 179 it would be discarded as expected, see connection test below. 

```
root@r2# run telnet 10.1.2.1 source 2.2.2.2 port 179
Trying 10.1.2.1...
telnet: connect to address 10.1.2.1: Connection refused
telnet: Unable to connect to remote host
```

As observed this is fairly straightforward to configure, but now let's say r1 needs to peer with 50 more routers within that BGP group, this clearly becomes cumbersome since we would need to add new IP addresses per BGP peer we configure. This is obviously prone to errors and/or potentially forgetting to add the new peer IP's under the prefix-list leaving the network unprotected. This where apply-path feature comes into play.

Instead of updating `prefix-lists` to match the BGP peer address every time a new peer is added, we create a *single* apply-path and inherit the peer IP address from the configuration automatically, this then is passed to `prefix-list` which will contain the list of peer IP's.

If we delete, add, rename IP's, the apply-path would be updated dynamically. 

#### Example 2 - Allowing r1 BGP peer, rejecting everything else with apply-path
```
# r1 configuration:

policy-options {
    prefix-list BGP-PEER {
        apply-path "protocols bgp group <*> neighbor <*>";
    }
    policy-statement ACCEPT {
        then accept;
    }
}
```
If we want to see what is the `apply-path` matching, the usual check won't work, instead, we would need to add the switch `display inheritance` in order to see what is being parsed. Verification of apply-path inheritance is shown below.

#### Example 3 - Checking prefix-list with apply-path
```
[edit]
root@r1# show policy-options prefix-list BGP-PEER | display inheritance
##
## apply-path was expanded to:
##     10.1.2.2/32;
##
apply-path "protocols bgp group <*> neighbor <*>";
```

If we add a new BGP peer in r1, the apply-path would be updated automatically.

```
[edit]
root@r1# copy protocols bgp group myPeerAS2 neighbor 10.1.2.2 to neighbor 10.1.2.3

[edit]
root@r1# show | compare
[edit protocols bgp group myPeerAS2]
      neighbor 10.1.2.2 { ... }
+     neighbor 10.1.2.3 {
+         description "to peer r3-AS2";
+         enforce-first-as;
+         import ACCEPT;
+         family inet {
+             unicast;
+         }
+         export ACCEPT;
+         remove-private {
+             all;
+         }
+         peer-as 2;
+     }

[edit]
root@r1# commit
commit complete
```

```
[edit]
root@r1# show policy-options prefix-list BGP-PEER | display inheritance
##
## apply-path was expanded to:
##     10.1.2.2/32;
##     10.1.2.3/32; << ! New prefix added automatically.
##
apply-path "protocols bgp group <*> neighbor <*>";
```

Beyond BGP we can use it to control protocol source IP addresses that connects to the Routing Engine (RE) such as TACACS, NTP, or SNMP by only allowing traffic from the IP addresses that are actually configured in the router.
