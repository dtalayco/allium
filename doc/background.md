# Terminology, Context and Background for Allium

## Overview

This document accumulates background related to networking as relevant
to Allium.

Defined terms are capitalized when used in their defined sense (e.g.
"a Managed Device" vs. "a device in general").

## Concepts

In general, Allium assumes packet switched networks of hosts, although
circuit switching systems (e.g. optical circuit switches) should be
addressed as well. See proposals.md, P-02.

We assume a basic familiarity with networking and so do not attempt to
define things in order or avoid all circularities.

A packet is a container for data. It is typically composed of headers
and data, but one device's header is another's data.

When transfered between devices, packets need Framing to be
delineated.

### Devices and Interfaces

Data is transfered between devices through **Interfaces**. We use the
term generically along with **Port**, but note that it is critical to
recognize that physical interfaces can be chopped up (virtualized into
sub interfaces by VLANs for example) or aggregated (for example
forming 802.3ad link aggregates).

We identify three rough classes of devices: switches, hosts and 
appliances. There is gray area here, but roughly:

* Hosts run processes that produce and consume packets.
* Switches form the substrate of the network connecting hosts through
multiple interfaces.
* Appliances are switches with specialized processes that may manipulate
packets.

In any case, all devices have one or more physical interfaces which 
are connected to other devices and through which packets are exchanged.

Allium is concerned with the devices it can inspect and control, so
we define:

* **Device**: A switch, host or appliance with network interfaces for which
Allium has an operational model and which Allium can, at least,
inspect to get state and metrics.
* **Managed Device**: A Device to which Allium can send commands (as
well as inspect for state and metrics).

If needed, we'll refer to a Device which only supports inspection
(not control or configuration) as **Unmanaged** or **RO**

To emphasize the fact that Allium is built around a driver model for
device support, a **Driver** is the Allium component that maps between
Allium's abstractions and a specific Device's control interface.

A **Capability** is a feature or abstraction that a Driver declares a
Device supports, such as a treatment type or a telemetry
counter. Allium uses Capabilities to decide what it can ask of a
Device.

We note up front that the schema that defines Capabilities is critical
and likely to determine the success and effectiveness of Allium in the
real world.

**Links** connect two interfaces, usually on different devices, with
bidirectional packet flow.

### Networks

A **Topology** is a set of Devices and the Links connecting
them. There are some templates, like spine-leaf, mesh and ring, but
Allium generally makes no assumption about the roles of positions in
these templates.

A **Network** is physical or logical Topology. Logical networks
require some protocol for defining the virtual connectivity
(e.g. VLANs, a routing instance, an encapsulation specification).

A **Virtual Network** is a logical Network built on top of another
Network. It has its own connectivity and Rules, and may, for example,
be delegated to a Tenant for administration.

**L2** (Layer 2) and **L3** (Layer 3): See the next section for a
discussion of routing distribution which is really the important
matter.  Otherwise, we use L2 and L3 to refer to two other attributes:
(1) Fields in a packet (L2 header, L3 header) whose meanings are
determined by which protocols are implemented; and (2) the addresses
used (MAC, IP, etc) to identify Interfaces on Devices across a
network.

**BGP** (Border Gateway Protocol): An example routing protocol that
exchanges reachability information. We use it as a canonical example
of how a software process runs on switches with the goal of
distributing forwarding maps (addresses to interfaces, possibly with
packet modifications implied) across a network of switches.

### Addressing, L2 and L3

The differences between L2 and L3 as addressing layers is only clear
once specific protocols are given as context. But what is more
important, and often not addressed by layer-defining standards, is
the mechanism by which address mappings are distributed to switches.
This was, in fact, the key operational difference between the layers.
The imposition of structure on L3 addresses ("network" and "host"
parts) added more options and complexity for forwarding logic, for
example, adding locational associations to addresses.

Originally, L2 address mappings were discovered when the owner of an
address generated a packet and the receiver "learned" the mapping of
the source address; L3 address mappings were usually distributed
between instances of protocol processes running on each switch and,
logically, communicating "out of band".

Since Allium is a controller, it is very concerned with these
mechanisms.

Finally, note that traditionally we refer to a particular
device's L2 or L3 forwarding tables (VRF, etc). But it is the
consistency of these tables between devices that results in proper
forwarding of traffic across a network.

### Flows and Treatment

**POP** (Point of Presence): An addressable entity on a network. This
is usually a process running on a Host, although a Switch may also
have POPs. Note that the "presence" implies a particular Network on
which to be present; hence the POPs on a switch are usually present on
a "management" network, although that is simply another Network
instance from Allium's point of view. 

It is easy and sometimes useful to identify a POP with a MAC or IP
address, for example. But here we want to allow for finer
granularity. The tuple including the L2 address, L3 address and L4
port number is often the best way to think about a POP in our model.

**Flow**: An ordered sequence of packets generated by one POP and destined
for one or more other POPs. The set of destinations is fixed for a
given Flow; a Flow with more than one destination is multicast.

As noted above, any given Device has its own set of forwarding tables.
Similarly, each Device will have state which maps a given packet to a
particular Flow based on local information. In the same way that
consistency between devices is responsible for proper forwarding
across a Network, consistent mapping of a given packet to a Flow
determines the success or failure of achieving a Treatment of a Flow
across a network.

**Flow Requirements**: What attributes are intended for the Flow. Examples
include latency, jitter and drop rate.

**Flow Class**: Often it's easier to identify all the Flows with the
same Flow Requirements together. These form a Flow Class, but that may
or may not be useful in our schema. TBD.

It should already be clear already that there is a tradeoff between
doing independent processing of a packet at each Switch versus allowing
some result of the processing of the packet to be conveyed between switches
along with the packet. There are common ways to **Mark** packets via
protocols that anticipate this need and define fields in their headers
to carry this information. Examples include DSCP and PCP.

This approach comes both with efficiency gains (one processing step is
done early and later ones may be done more easily by only inspecting
the Mark) but also has security implications: How do we establish the
boundaries of trust across which these values are respected?  Finally,
the approach does not alleviate the need to ensure consistency in
state across devices as the interpretation of a Marking on a packet by
a later device needs to be done with the same semantic the original
Device Marked the packet.

**SLO** (Service Level Objective): A measurable target for a Flow
Requirement, e.g. "99th percentile latency below 10µs" or "zero
drops".

**Treatment**: What actions are taken on a packet by a Device. This
includes being dropped, prioritized relative to other packets, and any
modifications to the packet.

Devices operating on packets do not have global knowledge. They have
only the packet itself, possibly some metadata (e.g. ingress port) and
the state of the Device's forwarding logic (e.g. tables) to associate
a packet to a Flow. Achieving consistent mapping to a Flow, then to a
Treatment is another instance of the state distribution problem
familiar from forwarding table distribution.

What we wish to achieve in the network is Treatment of packets during
their transit across multiple Devices resulting in the satisfaction of
the requirements for the packet's Flow. Thus, we view each Device as
applying a partial map from packet to the Treatment of the packet and
ask if the patching of these partial functions (across multiple
Devices) produces the desired Treatment of Flows across the network.

### Intent and Administration

We identify the following concepts for reference.

**Intent**: The desired state of the network as expressed by users, as
opposed to the device configuration that realizes it.

**Rule**: A scoped specification for the Treatment of packets. Rules are
hierarchical based on their Scope.

**Scope**: The part of the network to which a Rule or an administrative
right applies (e.g. a fabric, a pod, a Virtual Network). The precise
semantics are an open question (see proposals.md, P-05).

**Tenant**: A user group to which administration of part of the network
(e.g. a Virtual Network) is delegated.

**Intended State**: What the network should look like, according to
Intent (e.g. the planned cabling and configuration).

**Discovered State**: What Allium has observed about the network (e.g.
Devices and Links found via LLDP).

**Operational State**: The current runtime state of the network, such as
link status, counters and queue depths.

### Controller and Observability

**Controller**: The Allium software that holds Intent, gathers State,
manages Devices and provides services for APIs and the User Interface.

**Telemetry**: State and metrics collected from Devices (and possibly
Hosts), such as interface and queue counters, used to determine
whether SLOs are being met.

**Dashboard**: A user interface view that summarizes the state and
performance of the network or a part of it, such as a Topology view or
a utilization heat map.
