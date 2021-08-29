---
layout: post
title: "Junos vs. IOS: What makes an ABR an ABR?" 
slug: what-makes-an-ABR-an-ABR
---

Recently while navigating through the [Juniper Elevate](https://community.juniper.net) community I saw a [question](https://community.juniper.net/answers/communities/community-home/digestviewer/viewthread?GroupId=25&MessageKey=01b7b661-02fc-492d-8d4b-d5d3e7c99e4d&CommunityKey=18c17e96-c010-4653-84e4-f21341a8f208&tab=digestviewer&ReturnUrl=%2fbrowse%2fallrecentposts) that caught my interest around OSPF. After observing the behaviour I thought it would be good to create a blog on it since this is one of those minor implementation details between vendors that can have significant impact in reachability in certain scenarios, and in this blog we will explore that in sufficient detail.

What makes an ABR an ABR in OSPF? This is a question that may seem straightforward for many, but surprisingly the answer depends of whom you ask, and you will get a different answer whether you’re asking to a Cisco or a Juniper engineer. The single most accurate answer would always be: “An ABR is a router which Router LSA has the Border bit set”, while this is correct this does not tell us the full picture. IOS and Junos take a different approach when deciding when to set the Border bit:
- `IOS` would set the border bit when a router is attached to area 0 and any other area.
- `Junos` would set the border bit when a router is attached to two or more areas.

As seen, there is no strict requirement in Junos to be connected to area 0 in order for a router consider itself to be an ABR. This has implications in scenarios where we need an ABR to generate Type-3/NetSummary LSAs or when dealing with more advanced scenarios in Not-So-Stubby-Areas (NSSA) to determine which router would be elected as the Type-7-to-Type-5 LSA translator. We will explore an scenario that is interesting around the latter and that will be a good exercise for those who are from the Cisco world to observe the Junos behaviour in action, for those who are in the Junos world, stay around, since it might be something you might not expect. : )

#### Scenario: Why `vMX1` backbone router does not have the `4.4.4.4/32` route?
In this scenario we have vMX1 as an internal backbone router. vMX2 and vMX3 are ABRs connected to vMX4 with the areas set as NSSAs, while vMX4 is redistributing its connected `lo0.0` `4.4.4.4/32` with the policy referenced below `OSPF-REDIST`.

<img src="/assets/images/abr-post.png" alt="">

#### Initial configuration
```perl
vMX1:
jcluser@vMX1# show | match “ospf|interface” | display set
set interfaces ge-0/0/0 unit 0 family inet address 10.100.12.1/24
set interfaces ge-0/0/2 unit 0 family inet address 10.100.13.1/24
set interfaces lo0 unit 0 family inet address 1.1.1.1/32
set routing-options router-id 1.1.1.1
set protocols ospf area 0.0.0.0 interface lo0.0
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0

/

vMX2:
jcluser@vMX2# show | match “ospf|interface” | display set
set interfaces ge-0/0/0 unit 0 family inet address 10.100.12.2/24
set interfaces ge-0/0/2 unit 0 family inet address 10.100.24.1/24
set interfaces lo0 unit 0 family inet address 2.2.2.2/32
set routing-options router-id 2.2.2.2
set protocols ospf area 0.0.0.24 nssa
set protocols ospf area 0.0.0.24 interface ge-0/0/2.0
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf area 0.0.0.0 interface lo0.0

/

vMX3:
jcluser@vMX3# show | match “ospf|interface” | display set
set interfaces ge-0/0/0 unit 0 family inet address 10.100.34.1/24
set interfaces ge-0/0/2 unit 0 family inet address 10.100.13.2/24
set interfaces lo0 unit 0 family inet address 3.3.3.3/32
set routing-options router-id 3.3.3.3
set protocols ospf area 0.0.0.34 nssa
set protocols ospf area 0.0.0.34 interface ge-0/0/0.0
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0
set protocols ospf area 0.0.0.0 interface lo0.0

/

vMX4:
jcluser@vMX4# show | match “ospf|interface” | display set
set interfaces ge-0/0/0 unit 0 family inet address 10.100.34.2/24
set interfaces ge-0/0/2 unit 0 family inet address 10.100.24.2/24
set interfaces lo0 unit 0 family inet address 4.4.4.4/32
set routing-options router-id 4.4.4.4
set policy-options policy-statement OSPF-REDIST term LOOPBACK from interface lo0.0
set policy-options policy-statement OSPF-REDIST term LOOPBACK then accept
set protocols ospf area 0.0.0.24 nssa
set protocols ospf area 0.0.0.24 interface ge-0/0/2.0
set protocols ospf area 0.0.0.34 nssa
set protocols ospf area 0.0.0.34 interface ge-0/0/0.0
set protocols ospf export OSPF-REDIST
```
We can observe that vMX4 is indicating it is an ABR even though is not connected to area 0, it is also an ASBR since we are redistributing its `lo0.0` interface generating a Type-7/NSSA External LSA.

```perl
jcluser@vMX4# run show ospf overview
Instance: master
  Router ID: 4.4.4.4
  Area border router, AS boundary router, NSSA router <<< !
  Area: 0.0.0.24
    Stub type: Stub NSSA
    Area border routers: 1, AS boundary routers: 1
  Area: 0.0.0.34
    Stub type: Stub NSSA
    Area border routers: 1, AS boundary routers: 1
```

Expanding the router LSA in vMX4 we can observe the `bits 0x3` field in the LSA header, this corresponds to the ABR and ASBR bits being set, this is how the router via the Router LSA signals to the area that it is an ABR/ASBR for the NSSA.

```perl
jcluser@vMX4# run show ospf database router lsa-id 4.4.4.4 extensive

    OSPF database, Area 0.0.0.24
 Type       ID               Adv Rtr           Seq      Age  Opt  Cksum  Len
Router  *4.4.4.4          4.4.4.4          0x80000004   326  0x20 0xaa53  36
  bits 0x3, link count 1 <<< !
  id 10.100.24.2, data 10.100.24.2, Type Transit (2)
    Topology count: 0, Default metric: 1
  Topology default (ID 0)
    Type: Transit, Node ID: 10.100.24.2
      Metric: 1, Bidirectional
  Gen timer 00:25:56
  Aging timer 00:54:33
  Installed 00:05:26 ago, expires in 00:54:34, sent 00:05:26 ago
  Last changed 00:05:26 ago, Change count: 2, Ours

    OSPF database, Area 0.0.0.34
 Type       ID               Adv Rtr           Seq      Age  Opt  Cksum  Len
Router  *4.4.4.4          4.4.4.4          0x80000004   391  0x20 0x8762  36
  bits 0x3, link count 1 <<< !
  id 10.100.34.2, data 10.100.34.2, Type Transit (2)
    Topology count: 0, Default metric: 1
  Topology default (ID 0)
    Type: Transit, Node ID: 10.100.34.2
      Metric: 1, Bidirectional
  Gen timer 00:19:44
  Aging timer 00:53:28
  Installed 00:06:31 ago, expires in 00:53:29, sent 00:06:29 ago
  Last changed 00:06:31 ago, Change count: 2, Ours
```

vMX2 and vM3 is seeing this route towards vMX4 in the route table.

```perl
jcluser@vMX2# run show route table inet.0 4.4.4.4/32

inet.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

4.4.4.4/32         *[OSPF/150] 00:08:07, metric 0, tag 0
                    >  to 10.100.24.2 via ge-0/0/2.0

jcluser@vMX3# run show route table inet.0 4.4.4.4/32

inet.0: 16 destinations, 16 routes (16 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

4.4.4.4/32         *[OSPF/150] 00:09:34, metric 0, tag 0
                    >  to 10.100.34.2 via ge-0/0/0.0
```

vMX2 and vMX3 should translate this LSA on their respective areas from Type-7 to Type-5 (should they?), but while checking the route table and LSDB we see the route is no found nor the LSA coming from the ABRs.

```perl
jcluser@vMX1# run show route table inet.0 4.4.4.4/32
<no output>
jcluser@vMX1# run show ospf database external lsa-id 4.4.4.4
<no output>
```

The issue is quite clear, vMX4 is competing for the election of the NSSA translator, since it consider itself as an ABR and it has the highest router-ID it wins the election, effectively disallowing vMX2 and vMX3 from translating Type-7 to Type-5 LSAs. Note that this is a behavior we would never see in `IOS`, since as mentioned earlier `IOS` will only set the Border bit when a router is attached between area 0 and any other area. 
We can fix this by setting the router-ID in vMX4 to be lower than vMX2 and vMX3.

```perl
jcluser@vMX4# set routing-options router-id 1.1.1.4

jcluser@vMX4# commit
commit complete

jcluser@vMX4# run show ospf overview
Instance: master
  Router ID: 1.1.1.4 <<< !
  Area border router, AS boundary router, NSSA router
```

If we verify vMX1, we will see Type-5/External LSA coming from the ABRs and the route installed in the route table.

```perl
/LSA

jcluser@vMX1# run show ospf database external lsa-id 4.4.4.4 extensive
    OSPF AS SCOPE link state database
 Type       ID               Adv Rtr           Seq      Age  Opt  Cksum  Len
Extern   4.4.4.4          2.2.2.2          0x80000001    69  0x22 0x848d  36
  mask 255.255.255.255
  Topology default (ID 0)
    Type: 2, Metric: 0, Fwd addr: 10.100.24.2, Tag: 0.0.0.0
  Aging timer 00:58:50
  Installed 00:01:08 ago, expires in 00:58:51, sent 00:01:08 ago
  Last changed 00:01:08 ago, Change count: 1
Extern   4.4.4.4          3.3.3.3          0x80000001    74  0x22 0xe81b  36
  mask 255.255.255.255
  Topology default (ID 0)
    Type: 2, Metric: 0, Fwd addr: 10.100.34.2, Tag: 0.0.0.0
  Aging timer 00:58:46
  Installed 00:01:13 ago, expires in 00:58:46, sent 00:01:13 ago
  Last changed 00:01:13 ago, Change count: 1

/route table

jcluser@vMX1# run show route table inet.0 4.4.4.4/32

inet.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

4.4.4.4/32         *[OSPF/150] 00:00:06, metric 0, tag 0
                    >  to 10.100.12.2 via ge-0/0/0.0
                       to 10.100.13.2 via ge-0/0/2.0
```

We can ping!

```perl
jcluser@vMX1# run ping 4.4.4.4 rapid
PING 4.4.4.4 (4.4.4.4): 56 data bytes
!!!!!
--- 4.4.4.4 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.560/1.829/2.015/0.152
```

As observed, these behavioural differences that seem minor can lead to bigger and unexpected consequences in reachability.
