# Allium Requirements

## Overview

Allium is an open source network controller for SONiC switches, but
with a focus on clean abstractions independent of specific protocols
and implementations of them. The architecture supports a driver-like
model for adding support for new devices that support these
abstractions.

Any network controller is built on a set of abstractions of what a
network is and how it should work. This document is mostly about
defining that for Allium. These abstractions in turn drive the UI, UX
and APIs exposed by Allium. They also determine the interfaces that
need to be supported for a device to integrate with an Allium deployment.

Driven by AI deployments which generally bind end devices and their
interfaces tightly to network functionality, Allium leverages this and
provides an abstraction for edge devices (and interfaces). This allows
Allium to better address end-to-end traffic properties.

See background.md for definitions and background.

## Rules

High level scoped specifications for treatment of packets. Hierarchical
based on scope specification. (This puts a requirement on what "scope"
means and how it works.)

## Lifecycles

### Device

### Virtual networks

### Rules

## Operational Model

API required by Allium should drive device model requirements

Discovery: Devices, connections

Admission Policy:

User and Permission Model, controller side

Device access model: Drive towards "Allium Compliant Mgmt Interface",
but for now, ssh and command model for example

Network level abstractions: Physical Interface, Virtual Interface
(sub, grouped), L2, VLAN, L3, Host,

Rendering of state

Flow treatment attributes and specifications

Default flow treatment (best effort)

Mapping flows to flow treatments by using the operational model: devices,
interfaces, virtualized networks and interfaces; then packet props,

Measuring and reporting status for achieving specified flow treatment;
identifying violations of treatment specifications

UI

UX

## Goals

### Resilient controller

### Hierarchical administration model

Delegate rights for virtual networks and sub-networks.

### Network performance transparency

Should be easy to see if the network is satisfying its treatment
requirements. Failures for that should be clearly rendered and ideally
root caused.

### User interfaces exposing operational model

Graphical repr of topology

Graphical repr of current network utilization (heat map)

Allow the selection of a virtual network to display the above

Ability to apply policies manually or via automation

Extensible alerting mechanism and API/interface

