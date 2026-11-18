# Allium Proposals and Open Decision Points

This document is AI generated and has had limited review.

This document lists the open decisions facing Allium. For each one it
gives the context, the alternatives and their tradeoffs, and, where
there is one, a current *leaning*. A leaning is only a starting point
for discussion. It is not a decision.

Once a proposal is decided, it is recorded as an ADR in `doc/adr/`
and the entry here is reduced to a link (see [plan.md](plan.md)).

**Status values:** *Open* (not yet discussed), *Discussing*, *Leaning*
(tentative direction), *Decided* (moved to an ADR).

## Summary

| ID | Topic | Blocks | Priority | Status |
|---|---|---|---|---|
| P-01 | Initial target and scope | Requirements, everything | High | Open |
| P-02 | Circuit switching / optical in scope? | Glossary, model | Low | Open |
| P-03 | Host / edge participation | Model, drivers | High | Open |
| P-04 | How intent is expressed | Model, API, UI | High | Open |
| P-05 | Scope and rule hierarchy semantics | Model, compiler, authz | High | Open |
| P-06 | Flow identification (packet → flow) | Model, compiler | High | Open |
| P-07 | Treatment specification level | Model, drivers, telemetry | High | Open |
| P-08 | Intended vs. discovered authority | Reconciliation | Medium | Open |
| P-09 | Controller distribution / resilience | Architecture | Medium | Open |
| P-10 | State store / platform (incl. Kubernetes) | Architecture | Medium | Open |
| P-11 | Southbound protocol(s) | Drivers | Medium | Open |
| P-12 | Driver packaging and isolation | Drivers, ecosystem | Medium | Open |
| P-13 | Device identity and admission | Admission design | Medium | Open |
| P-14 | Config deployment semantics | Compiler, drivers | Medium | Open |
| P-15 | Telemetry and SLO measurement | Transparency goal | Medium | Open |
| P-16 | Northbound API style | API, CLI, UI | Medium | Open |
| P-17 | Implementation language | All code | Medium | Open |
| P-18 | Authorization and delegation model | Security, admin | Medium | Open |
| P-19 | Governance and contribution model | Community | Low | Open |
| P-20 | Glossary conventions | Docs | Low | Leaning (applied) |

---

## Scope

### P-01: Initial target and scope

**Context.** The requirements doc says "for SONiC switches" and also
"independent of specific protocols". It mentions AI deployments as the
motivating case. The v1 target determines scale assumptions, which
features are included, and which use cases come first.

**Alternatives.**

1. **AI/ML back-end fabric on SONiC.** Lossless or low-latency RDMA
   fabrics with spine-leaf topology, built on SONiC.
   - Pros: matches the stated motivation. Topology is regular and
     traffic classes are well understood (RoCE, PFC/ECN).
   - Pros: strong need for end-to-end treatment and host awareness.
   - Cons: narrow. It can make the general model look over-fitted.
2. **General data center fabric on SONiC.** L2/L3, VRFs and EVPN.
   - Pros: much broader audience.
   - Cons: large feature surface (EVPN/VXLAN), and many existing
     tools already compete here.
3. **Multi-vendor from day one.**
   - Pros: proves the abstractions early.
   - Cons: doubles the integration work before the model is stable.

**Tradeoffs.** A narrow target lets the project ship something real
sooner. The risk is that the abstractions get shaped too closely
around it. One mitigation: write the model doc against two very
different example networks, even if only one is implemented.

**Leaning.** Option 1 for the implementation. Validate the model on
paper against option 2's use cases.

**Questions.** What scale is v1 aimed at (tens, hundreds or thousands
of switches)? Is the front-end/storage network in scope alongside the
back-end fabric?

### P-02: Circuit switching and optical

**Context.** background.md says circuit switching "should be addressed
as well". Optical circuit switches are increasingly used in AI
fabrics.

**Alternatives.**

1. **Out of scope.**
2. **Model-compatible but not implemented.** Make sure links and
   topology can represent reconfigurable circuits.
3. **First-class support.**

**Leaning.** Option 2. Keep the topology model general enough that a
link can be dynamic, and defer all implementation.

### P-03: Host / edge participation

**Context.** One of Allium's distinguishing ideas is to treat hosts,
NICs and POPs as part of the network so it can deliver end-to-end
flow treatment. How far does Allium reach into the host?

**Alternatives.**

1. **Model-only.** Hosts appear in the model, populated from LLDP,
   ARP/ND or an inventory import. Allium configures nothing on them.
2. **Host agent, read-only.** An agent reports NIC state, counters and
   flow statistics.
3. **Host agent, managed.** The agent configures NIC QoS (DSCP/PCP
   mapping, PFC, DCQCN/ECN parameters) and possibly host-side marking
   of flows.
4. **Integrate with the existing host stack.** Rely on existing
   orchestration (Kubernetes CNI, schedulers) through an API, without
   an Allium agent.

**Tradeoffs.** Options 2 and 3 give the best end-to-end view and
control. They also expand the security footprint, operational burden
and platform matrix (OS, NIC vendor). Option 4 avoids agents but
depends on what external integrations expose.

**Leaning.** The model treats hosts as first-class from the start
(option 1). An agent comes later as an optional *driver*, using the
same driver model as switches, so a host is just another
device type.

---

## Abstraction Model

### P-04: How intent is expressed

**Context.** The design section "Expressing Intension" covers device
roles, connectivity/topology and flow treatment. The form intent takes
shapes the API, the UI and the compiler.

**Alternatives.**

1. **Declarative desired-state resources**, Kubernetes-style: typed
   objects such as `Fabric`, `VirtualNetwork`, `TreatmentPolicy` and
   `FlowClass`, with continuous reconciliation.
   - Pros: familiar to operators. Works well with GitOps. Diffs and
     dry-runs are easy.
   - Cons: expressing cross-cutting rules can get verbose.
2. **A policy language / DSL**, in the spirit of NetKAT, Datalog or
   Rego.
   - Pros: expressive. Supports formal reasoning, which fits the
     partial-map composition idea.
   - Cons: steep learning curve. The tooling and editor support has to
     be built.
3. **Imperative API**: "create VLAN X on ports Y".
   - Pros: simple to build.
   - Cons: puts Allium back at the device-config level of abstraction
     it is trying to rise above.
4. **Hybrid.** Declarative resources, with rule bodies written in a
   small, constrained match/treatment expression language.

**Leaning.** Option 4. Keep the expression language intentionally
small (match predicates plus treatment attributes) so conflicts can be
analyzed.

**Questions.** Is "Intension" in design.md meant in the logic sense
(defining sets by predicate)? If so, that supports predicate-defined
scopes (see P-05).

### P-05: Scope and rule hierarchy semantics

**Context.** Rules are "hierarchical based on scope specification".
Delegated administration is hierarchical too. Both depend on what a
scope is.

**Alternatives for what a scope is.**

1. **Tree of named containers**, such as fabric → pod → rack, or
   org → tenant → virtual network.
   - Pros: simple, and the structure maps directly onto delegation.
   - Cons: a single tree cannot express cross-cutting groupings.
2. **Predicate-defined sets**, e.g. labels or selectors as in
   Kubernetes.
   - Pros: flexible.
   - Cons: overlapping scopes make precedence hard.
3. **Hybrid.** An administrative tree governs ownership and
   delegation. Label selectors can be used within a node of the tree
   to target rules.

**Alternatives for resolving conflicts.**

1. **Most-specific scope wins.**
2. **Explicit priority** values.
3. **Parent constrains child.** A child may refine a parent's rules
   but may not violate them, as in delegation with guardrails.
4. **Reject conflicts at admission time** (static analysis).

**Tradeoffs.** Implicit precedence (1) is convenient, but it surprises
users when scopes overlap. Static rejection (4) is the safest, but it
requires decidable conflict detection, so the rule language has to
stay small.

**Leaning.** Scope model 3 combined with resolution rules 3 and 4.
Parents set bounds, children refine within them, and overlapping
same-level rules are rejected unless the user gives them an explicit
priority.

### P-06: Flow identification (mapping packets to flows)

**Context.** background.md notes that devices only see the packet,
some metadata and their local state. How does a flow's identity get
carried or reconstructed hop by hop?

**Alternatives.**

1. **Classify at every hop** on header fields (5-tuple, etc.).
   - Pros: needs no cooperation from hosts.
   - Cons: costly in TCAM. Breaks with encapsulation or encryption.
   - Cons: classifiers can differ between devices.
2. **Classify once at the edge, then carry a mark** (DSCP/PCP, VLAN,
   VRF, VNI). Core devices act only on the mark.
   - Pros: scalable and consistent across the network.
   - Cons: few mark bits are available. Allium has to manage trust
     boundaries.
3. **Host-marked.** The host or NIC sets the mark, and the network
   trusts or validates it.
   - Pros: the most accurate application intent.
   - Cons: requires host participation (P-03).
4. **Aggregate flow classes only.** Do not track individual flows.
   Treatments apply to classes, and individual flows are only sampled
   for telemetry.

**Leaning.** Options 2 and 4 by default, with option 3 when a host
driver is present. Write this down explicitly in the composition
model: core devices' partial maps are functions of the mark, and edge
devices are where flow → mark gets decided.

### P-07: Treatment specification level

**Context.** Users want to state outcomes such as latency, jitter and
loss. Devices implement mechanisms such as queues, schedulers,
PFC/ECN, policers and buffers.

**Alternatives.**

1. **Mechanism-level only.** Users choose queue, scheduler weight and
   PFC on/off directly.
   - Pros: predictable, and simple to compile.
   - Cons: little abstraction value.
2. **Outcome-level (SLO) only.** Allium derives the mechanisms.
   - Pros: the ideal from the user's point of view.
   - Cons: deriving mechanisms from outcomes is a hard research
     problem, and results depend on load.
3. **Named service classes with SLO targets.** Each class, such as
   "lossless-rdma", "low-latency" or "best-effort", is a vetted
   mapping to mechanisms. Each also carries measurable SLO targets
   that Allium monitors.

**Leaning.** Option 3. Allium does not *guarantee* outcomes from
mechanisms. It *configures* known-good mechanisms and *verifies*
outcomes through telemetry, which supports the transparency goal.
Admission control and resource accounting (the "resource allocations"
section of design.md) can be added later on top of the classes.

### P-08: Intended vs. discovered state authority

**Context.** design.md lists "Reconciling Discovered and Intended
Topology".

**Alternatives.**

1. **Intent is authoritative.** Discovered differences are drift.
   Allium alerts and, optionally, remediates.
2. **Discovery is authoritative.** Discovery builds the model, and
   intent annotates it.
3. **Per-attribute authority.** Cabling and topology are intended
   (deviations are wiring errors). Hosts and dynamic state are
   discovered. Each attribute is explicitly marked with its source of
   truth.

**Leaning.** Option 3, with three distinct layers kept in the model:
intended, discovered and operational. A "propose intent from
discovery" workflow handles brownfield onboarding.

---

## Architecture

### P-09: Controller distribution and resilience

**Context.** "Resilient controller" is a stated goal, and design.md
begins with "Distributed Controller".

**Alternatives.**

1. **Single active instance with a warm standby.**
2. **Clustered replicas with consensus** (Raft) over shared state.
   Work is partitioned by leader or by lease.
3. **Hierarchical or sharded controllers**, e.g. per pod with a global
   layer above.
4. **Headless-first.** Devices keep running on their last config, and
   the network never depends on the controller being up for
   forwarding. This principle is orthogonal to options 1–3.

**Tradeoffs.** Consensus gives correctness but adds operational
complexity. Sharding scales but makes cross-shard intent harder.

**Leaning.** Adopt headless-first as a hard requirement. Build option 2
on a proven consensus store (see P-10). Leave option 3 as a future
scaling path, and keep the internal APIs shard-friendly.

### P-10: State store and platform

**Alternatives.**

1. **Build on Kubernetes.** CRDs for intent, operators for
   reconciliation, etcd underneath.
   - Pros: HA, RBAC, a watch API, GitOps and a large tool ecosystem
     come for free.
   - Cons: requires a Kubernetes cluster to run the network. That is
     a chicken-and-egg problem when the cluster runs on this network.
   - Cons: CRD scale limits, and the semantics are tied to Kubernetes.
2. **Standalone, with embedded or external etcd.**
   - Pros: proven consensus and a watch API, without needing
     Kubernetes.
3. **Embedded Raft library plus a local KV store.**
   - Pros: ships as one binary. Full control.
   - Cons: more code to own.
4. **Relational DB (Postgres) plus leader election.**
   - Pros: rich queries and familiar operations.
   - Cons: Postgres HA is its own project.

**Leaning.** Option 2, with an API shaped so that a Kubernetes adapter
(CRDs mirroring the Allium API) can be added later. Telemetry goes to
a separate TSDB in any case (see P-15).

### P-11: Southbound protocol(s)

**Context.** requirements.md mentions a future "Allium Compliant Mgmt
Interface", with ssh/CLI for now. SONiC offers several management
paths.

**Alternatives.**

1. **gNMI/gNOI with OpenConfig models** (SONiC's telemetry and
   mgmt-framework).
   - Pros: an industry standard, multi-vendor, with streaming
     telemetry.
   - Cons: SONiC's OpenConfig coverage varies by build and feature.
2. **SONiC REST / mgmt-framework.**
3. **Direct config_db / Redis access.**
   - Pros: complete SONiC coverage.
   - Cons: SONiC-internal schema that changes between releases.
     Fragile.
4. **SSH plus CLI scraping.**
   - Pros: works everywhere.
   - Cons: brittle, slow and hard to make transactional.
5. **NETCONF/YANG.**

**Leaning.** Define the "Allium Compliant Management Interface" as a
*profile*: gNMI/gNOI plus a specific set of OpenConfig (and Allium
extension) paths, plus required telemetry subscriptions. The SONiC
driver implements the profile, falling back to config_db where
OpenConfig coverage is missing. The Phase 6 SONiC spike will confirm
or change this.

### P-12: Driver packaging and isolation

**Alternatives.**

1. **In-process plugins**, compiled in or loaded dynamically.
   - Pros: fast and simple.
   - Cons: a buggy driver can crash the controller. Language lock-in.
2. **Out-of-process drivers over gRPC**, in the style of
   Terraform/HashiCorp go-plugin or CSI.
   - Pros: isolation. Drivers can be written in any language. Clear
     versioned contract.
   - Cons: IPC overhead and more processes to manage.
3. **Drivers as remote agents on the device itself**, e.g. a SONiC
   container.
   - Pros: local access and scale-out.
   - Cons: deployment burden on every device.

**Leaning.** Option 2 for the contract. Bundle first-party drivers in
the same artifact for convenience. Option 3 becomes a deployment choice
for the same driver contract once it is defined.

### P-13: Device identity and admission

**Alternatives for identity.**

1. Serial number or chassis MAC, from inventory.
2. Certificate-based identity, ideally IEEE 802.1AR DevID or TPM-backed.
3. Operator-assigned name bound to one of the above.

**Alternatives for admission policy.**

1. Manual approval of each device.
2. Pre-registered allowlist (serial/MAC), auto-admitted on match.
3. Automatic admission by role inference from topology position.

**Related.** Is zero-touch provisioning (ONIE/SONiC ZTP) part of
Allium or external to it?

**Leaning.** Identity uses option 3: an operator name bound to a
serial/MAC in v1, with a path to certificate identity. Admission
supports policies 1 and 2 in v1. ZTP integration is external in v1,
with Allium providing the config.

### P-14: Config deployment semantics

**Alternatives.**

1. **Full replace** of device config each time.
   - Pros: idempotent, with no drift accumulation.
   - Cons: risk of disruption. Not every device supports it.
2. **Incremental diff.**
   - Pros: minimal change.
   - Cons: requires an accurate model of current state.
3. **Network-wide transactions** (all-or-nothing across devices).
   - Pros: consistent.
   - Cons: hard. Needs two-phase or rollback support on devices.
4. **Staged rollout**: canary first, then waves, with automated
   verification and rollback.

**Leaning.** Diff-based (option 2) with a mandatory dry-run/diff
preview. Order updates safely (e.g. make-before-break for treatment
changes). Add staged rollout (option 4) for fabric-wide changes.
Defer true network-wide transactions.

### P-15: Telemetry and SLO measurement

**Context.** The "network performance transparency" goal requires
knowing whether each flow class is meeting its SLO.

**Alternatives for data sources**, which can be combined:

1. Streaming device counters over gNMI: interface, queue, PFC, ECN
   and drops.
2. Sampled flows: sFlow/IPFIX.
3. In-band telemetry: INT/IFA, where supported.
4. Active probing between POPs, by host agents or switch-based probes.
5. Host-side measurement, e.g. RDMA/NIC counters or application
   reports.

**Alternatives for storage.** Prometheus-compatible
(VictoriaMetrics/Mimir), InfluxDB, ClickHouse.

**Tradeoffs.** Device counters are cheap but only *suggest* SLO
violations. Probing and host data measure the experience directly, but
they need agents.

**Leaning.** v1 uses sources 1 and 2 with a Prometheus-compatible
store. SLO indicators are defined per service class as derived
metrics. Source 4 arrives with the host driver (P-03). Alerting goes
through an extensible rule and notification layer, possibly
Alertmanager-compatible.

### P-16: Northbound API style

**Alternatives.**

1. **gRPC plus a REST gateway**, generated from one IDL.
2. **REST/OpenAPI**, resource-oriented.
3. **GraphQL.**
   - Pros: flexible queries over the topology graph.
   - Cons: harder to cache and authorize.
4. **Kubernetes-style resource API** with watch semantics.

**Leaning.** An API-first, resource-oriented design with watch
streams, defined in one IDL (protobuf) and exposed as both gRPC and
REST. The CLI and UI are clients of the public API only.

### P-17: Implementation language

**Alternatives.**

1. **Go.**
   - Pros: dominant in networking control planes; gNMI, OpenConfig
     (ygot), etcd and Kubernetes libraries are all native. Easy to
     contribute to.
   - Cons: a less expressive type system for the model and compiler.
2. **Rust.**
   - Pros: strong types and safety.
   - Cons: a thinner networking-control ecosystem. Higher bar for
     contributors.
3. **Python.**
   - Pros: SONiC tooling affinity and fast prototyping.
   - Cons: performance, and typing discipline at scale.
4. **Mixed.** Go for the controller, any language for drivers through
   P-12.

**Leaning.** Go for the controller (option 4). Python is acceptable
for Phase 6 spikes.

### P-18: Authorization and delegation model

**Context.** One goal is to "delegate rights for virtual networks and
sub-networks".

**Alternatives.**

1. **Flat RBAC.**
2. **Hierarchical RBAC** bound to scope-tree nodes. Roles are
   inherited downward.
3. **Relationship-based (ReBAC, Zanzibar-style).**
   - Pros: expressive delegation.
   - Cons: more machinery.

**Questions.** Is a tenant a hard isolation boundary, with separate
data visibility, or only an administrative convenience?

**Leaning.** Option 2, aligned with the scope tree from P-05. Defer
hard multi-tenancy (data isolation) to after v1, but keep a tenant
field in the model from the start.

---

## Project

### P-19: Governance and contribution model

**Alternatives.**

- **Contribution sign-off:** DCO sign-off, or a CLA.
- **Governance:** BDFL/maintainers now, foundation later (LF, CNCF,
  LFN), or a foundation from the start.
- **Decision records:** the proposals → ADR flow described in
  plan.md.

**Leaning.** Keep the MIT license. Use DCO, which has lower friction.
Maintainer-led governance to start, with the ADR process for
decisions. Revisit foundation membership if adoption grows. Note that
SONiC is under the Linux Foundation.

### P-20: Glossary conventions

**Minor decisions.**

- How the two "Device" definitions are resolved.
- Whether "Intension" is intentional.
- Whether "switch" includes routers (current text says yes).
- Spelling and capitalization conventions for defined terms.

**Leaning.** Capitalize defined terms when they are used in their
defined sense. Rename the second "Device" to "Modeled Device".

**Update.** `background.md` now follows this leaning. It uses
"Modeled Device", capitalizes defined terms and keeps routers under
"Switch". "Intension" in design.md is still open.
