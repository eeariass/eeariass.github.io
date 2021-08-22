---
layout: post
title: Recently cleared the JNCIP-SP 
slug: jncip-service-provider-cleared.md
---

<img src=" https://images.credly.com/size/680x680/images/57dae551-be95-4d64-a29c-947aa1ff7090/L_03_prof_JNCIP-SP.png" style="width:200px;height:200px"/>

In the last couple of months, been really into Junos studying for learning purposes and also trying to do their certification program for Service Provider Routing & Switching track. Recently passed the JNCIP-SP exam, which I have to say it took me longer than expected, due to the lack of a 'single source of truth'. Usually in my Cisco CCNP R&S days, I relied on the Cisco Press books, mainly the 'Foundation Learning Guides', for Juniper, my main source of study was the Juniper documentation directly, I did review User Guides (equivalent to the Configuration Guides in Cisco) for [almost] every technology/keyword listed in the exam. One thing I was particularly worried about was QoS, ironically it was the topic I got the highest score, others were OK-ish. As of now, have in target the JNCIE-SP exam, hoping I am able to at least do my first (and hopefully last) attempt at it.

This has been my first experience in a certification program outside Cisco, ever, and it has been smooth, exam questions are based on real-world scenarios and well crafted.

Will post new stuff here, in case I find something interesting during my JNCIE-SP studies.

```
hostname XR1-CUSTOMER
logging console debugging
!
interface GigabitEthernet0/0/0/1
 description XR2-ISP1
 ipv4 address 10.1.2.1 255.255.255.0
!
interface GigabitEthernet0/0/0/2
 description XR3-ISP2-ATTACKER
 ipv4 address 10.1.3.1 255.255.255.0
!
route-policy PASS-XR2
  pass
end-policy
!
route-policy PASS-XR3
  pass
end-policy
 !
!
router bgp 64512
 bgp router-id 1.1.1.1
 address-family ipv4 unicast
 !
 neighbor 10.1.2.2
  remote-as 64513
  description XR2-ISP1
  update-source GigabitEthernet0/0/0/1
  address-family ipv4 unicast
   route-policy PASS-XR2 in
   route-policy PASS-XR2 out
  !
 !
 neighbor 10.1.3.3
  remote-as 64666
  description XR3-ISP2-ATTACKER
  update-source GigabitEthernet0/0/0/2
  address-family ipv4 unicast
   route-policy PASS-XR3 in
   route-policy PASS-XR3 out
  !
 !
!
end
```
