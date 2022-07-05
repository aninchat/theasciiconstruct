---
title: "Juniper Apstra Part II - Designing a data center"
date: 2022-07-05
draft: true
tags: [juniper, apstra, vxlan, bgp, evpn, jncia-dc, jncis-dc, jncip-dc]
description:
---
In this post, we'll start designing different blocks for our data center deployment with Juniper Apstra. We'll look at how to design a rack, and build a data center template.
<!--more-->

## Introduction

In this post, we'll start designing the core blocks for our data center deployment with Juniper Apstra. We'll look at how to design a rack, and build a data center template (that is eventually fed into a blueprint).

Designing a rack is a fairly involved process. To be completely thorough, we're going to define new logical devices, create interface maps for these and then put it all together to build a rack. Naturally, some of these terms (logical devices, interface maps and so on) are Apstra specific - not to worry, we'll break it all down and understand what these are and how they are used. 

## Topology

Our topology is the same one we used for Part I of this series. We're going to be focusing on just the vQFX devices and leave out the Arista vEOS and Cisco N9Kv for now (multivendor via Apstra will be a later post).

![topo1](/images/juniper/juniper_apstra_2/juniper-apstra-part2-topo-1.jpg)

## Designing a data center rack

Let's go back to the workflow I'd described in the first post:

![workflow1](/images/juniper/juniper_apstra_2/onboard_and_build_rack_workflow.jpg)

This is what we're going to detail out here, and understand various components involved in building a rack. 

### Agents

To take action on any device (or collect any data from a device), the device must first be known to the system (Apstra, in this case) - it must be a part of the inventory. In Apstra, this is facilitated by something called as 'Agents'. 

Agents can be of two types:

1. On-box - these are agents that are installed directly on the network device itself.
2. Off-box - these are agents installed on the Apstra server, as a container, for a particular network device. 

A complete compatibility list (what network operating system supports what type of agent) is available in the official Apstra documentation, found [here](https://www.juniper.net/documentation/product/us/en/apstra). Navigate to the 'User Guide'. 

Juniper supports only the off-box agent as of Apstra 4.1, and detailed instructions regarding the initial configuration required for the off-box agent to be installed can be found [here](https://www.juniper.net/documentation/us/en/software/apstra4.1/apstra-user-guide/topics/task/agent-off-box-create.html) for Apstra 4.1. 

If you don't want to read the guide, here's what you need: a super-user that Apstra will use to login to the system, ssh and netconf over ssh, and finally reachability to the network device from Apstra.

```
{master:0}[edit]
root@vqfx# show system login 
user aninchat {
    uid 2000;
    class super-user;
    authentication {
        encrypted-password "$6$CR8rOdvC$fZiYRGtALKZNGU1hWCyoNg5fqxqyJlANacEwEfRPHLawUWzzpMLdMw1NpR5Z3//hPckVZbTjYFkCHPBdaKzxo."; ## SECRET-DATA
    }
}

{master:0}[edit]
root@vqfx# show system services 
ssh {
    root-login allow;
}
netconf {
    ssh;
}
```




### Logical Devices
### Interface Maps
### Putting it all together to build a rack
## Designing a data center template
## References
