---
title: "Juniper Apstra Part I - "
date: 2022-06-06
draft: true
tags: [juniper, apstra, vxlan, bgp, evpn, jncia-dc, jncis-dc, jncip-dc]
description:
---
description
<!--more-->
## Introduction

In this post, I'll introduce Juniper Apstra. We'll take a look at what a true IBN system is supposed to be, and why Apstra exemplifies this. We'll also explore graph databases, which is a data storing model that Apstra is based on.

Finally, we'll go through various Apstra components and learn how to build everything we need, to deploy a simple Data Center.

**As a disclaimer, I'd like to say that a lot of what I've written about IBN and IBN systems here, comes from reading papers and book(s) written by the fantastic Jeff Doyle. There is no intent to plagiarize his work.** 

## Apstra - the what and the why

Juniper Apstra is a **multivendor** solution for Data Center automation and insights. It falls under the general bracket of Intent Based Networks/Networking (IBN) - this is a term thrown around by many vendors today, but Apstra really exemplifies this and is essentially an IBN system. The goal is to maintain and validate the network as a whole, and not just individual components of the network. 

The network itself is driven by intent - intent that is provided by the end user, but converted into deployable network elements by Apstra. As a user, the intent is captured in the form of various inputs - how many racks in a DC, how many leafs per rack, how many (and what kind) of systems are connected to these leafs, what kind of redundancy is required per rack and so on. This is then converted into vendor specific configuration by Apstra (thus providing a level of abstraction), and pushed via netconf over ssh to the respective devices to bring your intent to life. 

In his book called 'IBN for Dummies', Jeff Doyle has a great statement about IBN systems - "self-operates, self-adjusts and self-corrects within the parameters of your expressed technical objective". What is the expressed technical objective? It is nothing but the intent.  

As an IBN system, Apstra provides/is:

1. Base automation and orchestration - conversion of intent into vendor specific configuration, eliminating any human error and conforming to reference designs built for different deployment models.
2. A single source of truth - this is important because you cannot have uniform and non-conflicting intent with multiple sources of truth. Idempotency can only exist with a single source of truth. In the case of Apstra, the 'Blueprint' is the source of truth.
3. Closed loop validation - this is continuous verification, in real time, of current network state with intended state, to flag any potential deviation. This eventually leads to self correction and self operation.
4. Self operation - Only through closed loop validation, can the network self correct or at minimum, alert the user that the network is out of compliance and provide possible remediation steps. 

These, as you may have guessed, are the main tiers of an IBN system as well. 

## Topology

This is the topology that we'll be working with for a majority of this series. Juniper Apstra is installed as a thin VM on ESXi, and is connected to all network devices via an OOB network. This is crucial to remember because Apstra only offers an OOB connection and nothing else as of today. 

There are three Juniper vQFXs acting as leafs, 2 vQFXs as spines, one Arista vEOS as a leaf and one Cisco NXOSv as a leaf. 

![topo1](/images/juniper/juniper_apstra_1/juniper-apstra-part1-topo-1.jpg)

## A brief look at databases

Databases are broadly divided into relational databases and non-relational databases. It is important to understand what a graph database is (since this is a founding pillar that Apstra was built on), and how it is different from relational databases. In this section, we'll take a brief look at relational databases and how they are structured, and then understand how graph databases are different and why they are ideal for network infrastructure.

### Relational databases

Relational databases (also called as SQL databases) are structured as tables (rows and columns). These are typically fixed schemas, and used for data that does not change very often. Let's take a simple example: we have a table called 'networkdevice' that is a database for network inventory.

There are several fields in this database that describe a network device - hostname, software version, management IP address. This can be visualized as so:

![relationaldb1](/images/juniper/juniper_apstra_1/relational_db_1)

Naturally, there are many other pieces of data that we might want to store or link to a network device. It doesn't make sense to have everything in one table itself. This is where relationships are built, and hence the term "relational" database.

For example, let's say we'd like to have some device specific settings like AAA enabled or not, site name or site ID, it's role in the network (spine/leaf/other) and so on. We could have another table for this called 'devicesettings' that has all of this information, and build a relationship between this table and our 'networkdevice' table. This is done using something called as 'foreign keys'. A foreign key is simply a column that uniquely identifies a row in another table.

Let's add another column in our 'networkdevice' table, and we'll call it 'device_id'. This is the foreign key that will connect our two tables together. Visually, this would be something like this:

![relationaldb2](/images/juniper/juniper_apstra_1/relational_db_2)

Relational databases, despite the name, are not great at capturing data that is connected. Because multiple tables need to be queried to get the final data, the computation becomes increasingly expensive. This is usually okay when only a low number of tables exist, but this is hardly the case in real world deployments - the number of tables in use, and related, will likely be quite high. 

### Graph databases

Graph databases are a type of NoSQL databases (non-relational databases), which are ideal for navigating connected data. They are essentially nodes linked via edges, where the edges are called 'relationships'. Relationships are treated as first class citizens - it has a relationship first approach to storing and querying data. 

One of the biggest factors for using graph databases for network infrastructure is that graphs make it very easy to do dependency analysis and they are very good at finding out indirect relationships - what is the impact of shutting a link, bringing down a node, making a policy change and so on. A natural extension of this is that RCA is greatly improved - feeding a graph into machine learning pipelines generates far better outcomes and decisions as compared to feeding in isolated data sets. 

For the scope of our discussions here, we will look at something called as labeled property graphs. Property graphs allow you to store properties against a node, as well as the relationship itself. These are stored in a 'key:value' format. They are also called as 'labeled' property graphs because you can assign labels to a node, describing the 'type' of the node. 

Visually, this looks like so:

![graphdb1](/images/juniper/juniper_apstra_1/graph_db_1)

I didn't want to reinvent the wheel, so the above is just a stripped down version from a data center deployment (which we'll look at) in Apstra. I've just focused on two nodes and the relationship between them. 

The nodes here are a 'system' and an 'interface'. The relationship between them is 'hosted interfaces', essentially implying that the system (with it's listed properties) has an interface (with it's listed properties). 

As you can see, the 'system' node has 'key:value' properties assigned to it - these include a base identifier called 'id', the name of the system ('spine1', in this case), it's role in the DC deployment and so on. Similarly, the 'interface' node has some 'key:value' properties assigned to it as well - again, an identifier, the interface name, IPv4 address and so on.

Finally, the relationship has some 'key:value' properties as well, largely to specify the direction, indicated via a source ID (which matches the 'system' node) and a destination ID (which matches the 'interface' node).

These are just two nodes that I've shown as an example - the entire blueprint that is deployed in Apstra is instantiated as a graph that can be traversed. We'll take a more detailed look at it once we actually deploy a blueprint in later posts.

## Navigating the GUI

This is your first look at Juniper Apstra (at least the first within the context of this blog post). 

![gui1](/images/juniper/juniper_apstra_1/gui_first_look)

On this main page, a simple workflow is provided - build racks, design the network, create and deploy blueprint. Of course, it is a little more involved than this and we're going to break it down in this post, and several others that will follow.

On the left hand side, there is a pane with several elements. A lot of the day-0 work is done within the 'Devices', 'Design' and 'Resources' tab. In this post, we're going to focus on this only. 

The 'Devices' tab has the following options:

![gui2](/images/juniper/juniper_apstra_1/gui_devices_tab)

Here, you can create agents for devices that need to be onboarded (we'll cover this next), create agent profiles, look at device profiles and so on.

The 'Design' tab has the following options:

![gui3](/images/juniper/juniper_apstra_1/gui_design_tab)

'Design' is where you start to sculpture what your DC will look like. We'll look at this in more detail shortly. 

Finally, 'Resources' are fairly straightforward:

![gui4](/images/juniper/juniper_apstra_1/gui_resources_tab)

These are various resources that Apstra will use to automate your DC deployment. 

## General workflow within Apstra



## Agents

One of the core constructs in Apstra is the 'Device Agent' - this is how the inventory is built and maintained within Apstra. Agents can be both on-box (the agent is installed on the network device itself) and off-box (the agent is installed as a container within Apstra and it talks to the network device via ssh/api), depending on what the network device supports. 


## Resources
## Design
## Blueprint
## Staging a DC
## Deploying a DC
## References

1. [IBN for Dummies](https://www.juniper.net/content/dam/www/assets/ebooks/us/en/intent-based-networking-for-dummies.pdf) by Jeff Doyle.
2. 


