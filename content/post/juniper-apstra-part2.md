---
title: "Juniper Apstra Part II - Building a data center rack"
date: 2022-07-05
draft: true
tags: [juniper, apstra, vxlan, bgp, evpn, jncia-dc, jncis-dc, jncip-dc]
description:
---
In this post, we'll start designing the building blocks for our data center deployment with Juniper Apstra. We'll look at how to design a rack, and build a data center template.
<!--more-->

## Introduction

In this post, we'll start designing the building blocks for our data center deployment with Juniper Apstra. We'll look at how to design a rack, and build a data center template (that is eventually fed into a blueprint).

Designing a rack is a fairly involved process. To be completely thorough, we're going to define new logical devices, create interface maps for these and then put it all together to build a rack. Naturally, some of these terms (logical devices, interface maps and so on) are Apstra specific - not to worry, we'll break it all down and understand what these are and how they are used. 

## Topology

Our topology is the same one we used for [Part I](https://www.theasciiconstruct.com/post/juniper-apstra-part1/) of this series. We're going to be focusing on just the vQFX devices and leave out the Arista vEOS and Cisco N9Kv for now (multivendor via Apstra will be a later post).

![topo1](/images/juniper/juniper_apstra_2/juniper-apstra-part2-topo-1.jpg)

## Designing a data center rack

**Our reference for this entire process is based on Apstra 4.1.0.**

Let's go back to the workflow we'd described in the first post:

![workflow1](/images/juniper/juniper_apstra_2/onboard_and_build_rack_workflow.jpg)

This is what we're going to detail out here, and understand various components involved in building a rack. 

### Agents and Agent Profiles

To take action on any device (or collect data from any device), the device must first be known to the system (Apstra, in this case) - it must be a part of the inventory. In Apstra, this is facilitated by something called as 'Agents'. 

Agents can be of two types:

1. On-box - these are agents that are installed directly on the network device itself (applicable only if supported by the network operating system).
2. Off-box - these are agents installed on the Apstra server, as a container, for a particular network device. 

A complete compatibility list (what network operating system supports what type of agent) is available in the official Apstra documentation, found [here](https://www.juniper.net/documentation/product/us/en/apstra). Navigate to the 'User Guide'. 

Juniper supports only the off-box agent as of Apstra 4.1.0, and detailed instructions regarding the initial configuration required for the off-box agent to be installed can be found [here](https://www.juniper.net/documentation/us/en/software/apstra4.1/apstra-user-guide/topics/task/agent-off-box-create.html) for Apstra 4.1. 

If you don't want to read the guide, here's a quick summary: a super-user that Apstra will use to login to the system, ssh and netconf over ssh, and finally reachability to the network device from Apstra.

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

As part of the device discovery, some basic parameters are needed, which are a part of constructing the Agent itself - this includes the platform (junos/eos/nxos), the IP address to reach the device and the username/password for logging into the device. 

![offbox1](/images/juniper/juniper_apstra_2/offbox_agent_parameters.jpg)

As you can imagine, this can get pretty tedious if you have to choose the platform and type in the username/password every single time for every new device that needs to be onboarded. This is where **Agent Profiles** are useful - you can create a profile for a specific type of device and just use the profile whenever that type of network device needs to be onboarded.

As an example, let's create an Agent Profile for my vQFX devices that need to be onboarded:

![offbox2](/images/juniper/juniper_apstra_2/offbox_agent_profile_vqfx.jpg)

You can see that I also have similar profiles created for Arista vEOS and Cisco N9Kv:

![offbox3](/images/juniper/juniper_apstra_2/offbox_agent_profile_others.jpg)

Now, I can type in the IP addresses of the devices I want to onboard (my two vQFX spines, and three vQFX leafs, in this case) and choose the vQFX profile.

![onboardvqfx](/images/juniper/juniper_apstra_2/onboarding_vqfx.jpg)

These devices are now listed under 'Managed Devices' under the main 'Devices' tab. While Apstra makes contact with the devices that need to be onboarded, and collect data that it needs to onboard it, you can view the logs for any device that is being onboarded (or was onboarded already). 

Just click on the button with the three dots (extreme right of each device row), and you get a set of actions available to you for that particular device. One of these actions is 'Show Log'.

![showlog1](/images/juniper/juniper_apstra_2/show_log_option.jpg)

The logs from a successful onboard of a vQFX device looks something like this:

![showlog2](/images/juniper/juniper_apstra_2/logs_vqfx_onboarding.jpg)

As part of the process, Apstra collects the configuration from the device and stores it as the pristine configuration - this is the baseline configuration without any services and features added to the device, apart from what was needed to get the device onboarded into Apstra. The Apstra state for the device at this point is called 'OOS-QUARANTINED' where 'OOS' stands for 'Out Of Service'.

The entire device lifecycle can be found [here](https://www.juniper.net/documentation/us/en/software/apstra4.1/apstra-user-guide/topics/topic-map/device-config-lifecycle.html). Once a device is onboarded, it needs to be explicitly acknowledged by Apstra - this acknowledgment kicks off the 'Discovery 1' process where LLDP is enabled on all interfaces. You can acknowledge a device by selecting the device (or selecting a subset of devices) and clicking on the 'acknowledge selected systems' button, and a subsequent confirmation:

![ack](/images/juniper/juniper_apstra_2/ack_devices.jpg)

![confirmack](/images/juniper/juniper_apstra_2/confirm_ack.jpg)

At this point, connected devices should see each other as LLDP neighbors. The Apstra state for the device moves from 'OOS-QUARANTINED' to 'OOS-READY' which essentially implies that the device is out of service, but it is ready and can now be added into a blueprint.

### Logical Devices

Logical Devices, as the name suggests, are just a logical or abstracted representation of your network device. The important thing to remember is that this is not vendor specific at all - you're not choosing Cisco or Arista or Juniper with a logical device. You're simply stating that you'd like a device that has, for example, 24x10G ports, with 4x100G uplinks. Or whatever other speed/port combination you're looking for.

A logical device also allows you to specify what kind of devices these ports can be connected to and used for - again, for example, you'd like your 4x100G uplinks to be used for connections to a spine, and your 24x10G ports as connections to servers.

Logical devices can be found under 'Design'. There are numerous logical devices that are already pre-built, covering a large range of existing (and supported) network devices from different vendors such as Cisco, Arista and Juniper. 

![logical_device_main](/images/juniper/juniper_apstra_2/logical_device_main_page.jpg)

We're going to create two new logical devices to go through the workflow: one for a vQFX leaf, and another for a vQFX spine. We're also going to be specific with what ports can connect to what network devices.

We're going to have 6 ports available on the leaf (10G each) to connect to spines:

![vqfx_leaf_ld1](/images/juniper/juniper_apstra_2/vqfx_leaf_ld1.jpg)

Click on the 'Create Port Group' option to create a port group for these 6x10G ports. 

![vqfx_leaf_ld2](/images/juniper/juniper_apstra_2/vqfx_leaf_ld2.jpg)

You can continue to add more ports by dragging the port group further. We're going to add another 6 ports that can be used to connect to the hosts. Hosts can be selected as 'Generic':

![vqfx_leaf_ld3](/images/juniper/juniper_apstra_2/vqfx_leaf_ld3.jpg)

You should now have a 12x10G vQFX logical device:

![vqfx_leaf_ld4](/images/juniper/juniper_apstra_2/vqfx_leaf_ld4.jpg)

Click on 'Create' to create this logical device. We're going to do the same thing for our vQFX spines, but all ports are going to be marked as connected to leafs.

![vqfx_spine_ld1](/images/juniper/juniper_apstra_2/vqfx_spine_ld1.jpg)

Create the port group, and click on create just like before to build this vQFX spine logical device:

![vqfx_spine_ld2](/images/juniper/juniper_apstra_2/vqfx_spine_ld2.jpg)

### Device Profiles

Device Profiles are where the actual hardware is defined. This includes (among other things): 

- what kind of vendor specific model the hardware is (PID) 
- the network operating system 
- what versions to match on (based on a regular expression) 
- CPU and RAM on the device 
- the kind of ASIC it supports
- CoPP support
- interfaces, the kind of connector type for it, and the naming convention

For vQFX (as well as vEOS and N9Kv), a device profile already exists, so we're not going to create a new one. But I'd still like to walk you through some of these options to show you how it can be done, particularly how the ports are defined via "transformations".

Device Profiles can be found under 'Devices'. The main page for this lists out all of the available device profiles, with a search bar to filter as you wish.

![device_profile_main](/images/juniper/juniper_apstra_2/device_profile_main.jpg)

If you'd like to create a new profile, you're met with the following page and options:

![new_device_profile1](/images/juniper/juniper_apstra_2/new_device_profile1.jpg)

The selector option lets you specific more vendor specific information:

![new_device_profile2](/images/juniper/juniper_apstra_2/new_device_profile2.jpg)

I want to focus on the 'Ports' section for a little bit, as that is very interesting. Here, you're greeted with a famiilar looking UI (this is similar to how we created logical devices). By default, one panel is present which is what you'd have for a non-modular network device. Considering our vQFX example, this panel should have 12 ports:

![new_device_profile3](/images/juniper/juniper_apstra_2/new_device_profile3.jpg)

You can now select a port, a subset of ports or all the ports to associate properties against them, like the connector type (SFP, QSFP and so on). For example, I have selected port 1 (by simply clicking on it):

![new_device_profile4](/images/juniper/juniper_apstra_2/new_device_profile4.jpg)

You also need to associate a "transformation" against the port. This is how the interface naming is actually generated (which, as you can imagine, can be vastly different from vendor to vendor), as well as the speed of the interface. Cliking on 'Add new transformation' lets you do this. 

![new_device_profile5](/images/juniper/juniper_apstra_2/new_device_profile5.jpg)

Before we move on from this, I'd like to show you the pre-built vQFX device profile:

![vqfx_device_profile](/images/juniper/juniper_apstra_2/vqfx_device_profile.jpg)

These are 12x10G ports, with the transformation defining the interface naming as xe-0/0/0 for port 1 and so on, which is what we expect. 

### Interface Maps

Interface Maps are fairly straightforward, but extremely important. A logical device and a device profile feed into it, and are used to create an interface map. This is where your logical abstraction of a device, and it's hardware specific information all comes together to form an actual deployable network device in your blueprint (which we'll talk about in the next post).

![interface_map_workflow](/images/juniper/juniper_apstra_2/interface_map_workflow.jpg)

From the UI, this is found under 'Design' again. Like I said, an interface map just takes in two feeds - a logical device, and a device profile.

![interface_map1](/images/juniper/juniper_apstra_2/interface_map1.jpg)

For our case, we're going to create two interface maps - one for the vQFX leaf and another for the vQFX spine. Starting with the leaf, we have our leaf logical device and the vQFX device profile as the input. Once this is done, there is some additional selection that needs to be done - this is the port selection for the port groups that were created within the logical device:

![interface_map2](/images/juniper/juniper_apstra_2/interface_map2.jpg)

The 'Select Interface' is kind of a drop down, which when clicked, lists all the available ports that can be selected (and mapped using the device profile) for that port group.

![interface_map3](/images/juniper/juniper_apstra_2/interface_map3.jpg)

This can be a little confusing, since it might seem like we did this exercise while creating a logical device as well, but think of it as this way - with the logical device, you're only defining the number of ports, their speeds, and what they are allowed to be connected to. In the interface map (and using the device profile), you're actually selecting which ports can do what and mapping them to a vendors standard naming convention (using a transformation that was defined in the device profile) for that kind of port.

So, in our case, I'd like the first 6 ports to be available as my spine connections, and use transformation #1 which was the default transformation present in the pre-built vQFX device profile. Let's select those:

![interface_map4](/images/juniper/juniper_apstra_2/interface_map4.jpg)

The remaining 6 ports are now chosen for my host/server connections:

![interface_map5](/images/juniper/juniper_apstra_2/interface_map5.jpg)

Finally, as a confirmation, if you scroll down a bit during the interface map creation, you can select any port to see a summary of what it would look like when actually deployed. I've selected port 1, which tells me it is port #1 of panel #1, and it is named xe-0/0/0.

![interface_map6](/images/juniper/juniper_apstra_2/interface_map6.jpg)

For our spine interface map, a similar process can be followed. In this case, there is just one port group with all 12 ports, and we select all of them with the same default transformation #1 from the pre-built vQFX device profile.

![interface_map7](/images/juniper/juniper_apstra_2/interface_map7.jpg)

### Putting it all together to build a rack
## Designing a data center template
## References
