# Allium Architecture

Work in progress

The document is intended only to have placeholders for the parts of the
architecture that are known or for which active work is underway.

## Overall Architecture

* **Drivers** connect to devices
* **Modules** instantiate Allium network abstractions

Example modules

* An admission module that runs the discovery protocol and applies the
admission criteria and owner assignment for new devices.
* A virtual network definition module that accepts user or API requests to create a
network, along with parameters necessary to instantiate it, checks for consistency
relative to existing state, then updates the internal state to reflect the change.
* Flow treatment spec module. Expose an API or UI to specify how to identify a flow
and the intended treatment of that flow.
* A circuit switching module which is given access to a set of circuit
switching devices and access to hosts or other schedule generating devices
that drive the circuit lifecycle.

Open considerations

* Resiliency
* Device Lifecycle
* Glue connecting drivers to modules
* Users and rights model

## The naming of cats

Allium needs a clean way to represent device instances to the UI and refer to those
internally in a consistent way.



