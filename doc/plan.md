# Allium Project Plan

## Purpose

This document lays out how Allium goes from a set of skeleton documents
to a working open source network controller. The project starts
**documents first**. Concepts, requirements, the abstraction model and
the architecture are written down and reviewed before code is built.
Code written early is limited to small, throwaway prototypes that
answer specific design questions.

Open decisions are tracked in [proposals.md](proposals.md). Each phase
below lists the proposals it has to resolve before it can finish.

## Status and Next Steps

*Last updated: 2026-11-18. Update this section before switching
machines or ending a work session, so the working context travels with
the repo.*

**Current phase:** Phase 1 (Concepts and Terminology), mostly done.
Phase 2 (Requirements) is next. Work is on the `towards_requirements1`
branch.

**Done recently**
- `background.md`:
  - Rewritten around the Thesis: correct behavior depends on
    consistency of state across devices.
  - Two rounds of review applied.
  - The partial-map paragraph restored.
- `architecture.md`: started, with the Driver/Module split, example
  modules and "The naming of cats".
- `proposals.md`: P-21–P-25 added. P-02, P-13 and P-20 revised.
- `plan.md`: updated for the above.

**In progress / open threads**
- Whether "Intension" in `design.md` is intentional (P-20).
- Whether the circuit switching module in `architecture.md` is a v1
  goal or a validation case for the Module design (P-02).
- Splitting `design.md` and `architecture.md`. Proposal:
  `architecture.md` is the overview, and `design.md` becomes an index
  of subsystem designs.

**Next actions**
1. Formalize the partial-map composition model (Phase 1).
2. Write the prior-art section (Phase 1).
3. Make the first decisions: P-01, P-03, P-04, P-05, P-21, then P-22
   and P-23.
4. Draft personas and use cases for `requirements.md` (Phase 2).
5. Phase 0 housekeeping: README, CONTRIBUTING, `doc/adr/` template.

**Resuming with an AI assistant:** start with "Read doc/plan.md
(especially this section), doc/proposals.md and doc/background.md;
we're continuing the design work."

## Where the Docs Stand Today

### What exists

| Document | State | Strongest content |
|---|---|---|
| `README.md` | Placeholder | — |
| `doc/background.md` | Thesis plus a concept-driven glossary | Thesis: correct behavior = consistency of state across devices; the "partial map" view of packets → treatment |
| `doc/requirements.md` | Outline with some prose | Edge/host-inclusive abstraction for AI deployments; hierarchical rules; performance transparency |
| `doc/architecture.md` | Early outline | Drivers vs. Modules split; example modules; naming of device instances |
| `doc/design.md` | Headings only | Section list gives a reasonable design outline |

### Core ideas already present (to preserve and develop)

0. **Consistency of distributed state (the Thesis).** Each device acts
   only on local information, but correct network behavior depends on
   the state being consistent across devices. That covers forwarding,
   mapping packets to flows, interpreting marks, and what a treatment
   means on each device. Allium's job is to establish and verify that
   consistency. The ideas below follow from this one.
1. **Abstraction first, SONiC first.** SONiC is the first target, but
   the model must not depend on any one protocol or implementation.
   New devices are added through a driver-like model.
2. **The edge is part of the network.** Hosts, NICs and POPs are
   first-class objects. The goal is end-to-end treatment, which goes
   beyond configuring switches one at a time.
3. **Flows and treatments, not just config.** Users state the
   requirements a flow has (latency, jitter, loss). Allium maps them
   to per-device treatments. It then measures whether the combined
   result meets the requirements.
4. **Composition of partial maps.** Each device applies a partial
   function from (packet, metadata, local state) to a treatment. The
   network-level question is whether chaining these partial functions
   together produces the intended per-flow treatment. This framing
   could become Allium's main intellectual contribution. It should be
   formalized early.
5. **Hierarchical scope.** Rules, administration and delegation are all
   hierarchical. A single definition of "scope" has to serve all
   three.
6. **Transparency.** Users should be able to see whether the network
   is meeting its commitments. When it is not, Allium should root-cause
   the failure.

### Gaps and issues found in review

- **background.md:** the first-round issues have been addressed: the
  duplicate "Device" definition, placeholders, missing terms and
  typos. "Device" now means a device Allium has a model for, and a
  device that can only be inspected is an "RO" Device. Still open:
  - Path selection as part of Treatment ("left for further study";
    see P-25).
  - Management network and bootstrap (deferred to a separate document;
    see P-24).
- **architecture.md**
  - The key open item is how Drivers and Modules connect (see P-23).
  - The circuit switching module implies more circuit-switching scope
    than P-02 currently leans toward.
  - "The naming of cats" needs an identifier scheme (see P-13).
- **requirements.md**
  - No personas or use cases. It does not say who operates Allium or
    what deployment it targets: a single AI cluster, a multi-tenant
    data center, or a campus.
  - No non-functional requirements: scale (number of devices,
    interfaces, flows), convergence time, controller HA targets, or
    behavior when the controller is unavailable.
  - No non-goals. Two examples of things to rule in or out: circuit
    switching and optical, and WAN.
  - The Lifecycles section is empty.
  - "Rules" points at a definition of scope that does not exist yet.
  - Requirements and design are mixed together. The Device access
    model section, for example, already picks ssh.
  - The requirements are not numbered, so later documents cannot trace
    back to them.
- **design.md**
  - Overlaps with architecture.md. Proposed split: architecture.md
    covers the component structure; design.md becomes an index of
    the subsystem designs under `design/`.
  - Headings only. "Expressing Intension" may be a typo for
    "Intention", or it may deliberately use the logic term
    *intension* (defining a set by its properties rather than by
    listing its members). The second reading fits the rule and scope
    concept well. Pick one and say which.
  - Missing sections: overall architecture and components, state
    store, southbound driver interface, northbound API, telemetry
    pipeline, config rendering/deployment, security, upgrade.

## Proposed Document Set

The existing documents are too few to hold the design. The
proposed set is below. Each document should be small enough to review
in a single sitting.

```
README.md                  Project pitch, status, links
CONTRIBUTING.md            How to propose changes, review process
GOVERNANCE.md              Decision-making, maintainers (later)
doc/
  plan.md                  This document
  proposals.md             Open decision points (index + summaries)
  background.md            Terminology & concepts (glossary)
  requirements.md          Numbered requirements, use cases, non-goals
  model/
    overview.md            The Allium abstraction/information model
    flows-and-treatment.md Flow identity, treatment, SLOs, composition
    scope-and-rules.md     Scope semantics, rule hierarchy, conflicts
    lifecycles.md          Device, network, rule lifecycles (state machines)
    capabilities.md        Capability schema (what Drivers declare)
  architecture.md          Architecture overview (Drivers, Modules, data flow)
  design.md                Index of subsystem designs
  design/
    controller.md          Distribution, HA, state store
    southbound.md          Driver model, device interface, SONiC driver
    management-network.md  Allium's own management network, bootstrap
    naming.md              Device/instance identifiers and naming
    northbound.md          API, CLI, UI contract
    discovery-admission.md Identification, discovery, admission
    reconciliation.md      Intended vs. discovered vs. operational state
    telemetry.md           Collection, storage, SLO evaluation, alerting
    security.md            Authn/authz, delegation, device trust
  adr/                     Accepted decisions (one file per decision)
```

**Decision workflow:** a decision point starts as an entry in
`proposals.md`. Once it is discussed and decided, it moves into
`doc/adr/NNNN-title.md` with status *Accepted*. The proposals entry is
then reduced to a link to the ADR.

## Phases

The phases are ordered by dependency, but they overlap in practice.
Each phase ends at a review gate: its documents are reviewed and
merged, and the decisions it lists are recorded as ADRs.

### Phase 0 — Project Foundations

Goal: make it easy for others to understand the project and contribute.

- [ ] Expand `README.md` with a vision paragraph, project status
      ("design phase") and links to the docs.
- [ ] Add `CONTRIBUTING.md` covering the doc review process, the
      proposal → ADR workflow, and DCO or CLA (see P-19).
- [ ] Create `doc/adr/` with a template and ADR-0001, "Record
      architecture decisions".
- [ ] Set up issue templates for the *proposal*, *question* and
      *doc bug* types.
- [ ] Optional: a Markdown lint/link-check CI job, so docs stay
      healthy from day one.

Decisions: P-19 (governance and contribution model).

### Phase 1 — Concepts and Terminology

Goal: a shared vocabulary that every later document can rely on.

- [x] Fix the inconsistencies, typos and placeholders in
      `background.md` (see the review findings above).
- [x] Add the missing terms. Distinguish *intended*, *discovered* and
      *operational* state.
- [x] Add a Thesis section to `background.md`.
- [ ] Turn the partial-map paragraph at the end of "Flows and
      Treatment" in `background.md` into a short, semi-formal statement
      of the composition model:
      - device treatment function: `T_d : (pkt, meta, state_d) → (treatment, pkt', egress)`
      - path composition and the conditions under which it satisfies a
        flow requirement
      - what can and cannot be verified with local information only
- [ ] Write a "prior art" section and say how Allium differs from each
      project. Candidates: ONOS, OpenDaylight, Faucet, Tungsten Fabric,
      SONiC-based fabric managers, Kubernetes network operators,
      intent-based networking work (IETF/IRTF NMRG), NetKAT and
      other network programming languages.

Decisions: P-02 (circuit/optical scope), P-20 (glossary conventions,
minor).

### Phase 2 — Requirements

Goal: a numbered, testable requirements set with clear scope.

- [ ] Personas, for example: fabric operator, tenant/ML-platform admin,
      SRE/on-call, automation (CI/CD, schedulers).
- [ ] 5–10 concrete use cases written as end-to-end narratives.
      Examples:
      - Bring up a new spine-leaf AI fabric from bare SONiC switches.
      - Give a training job a lossless, low-latency network slice
        across 64 hosts.
      - Delegate a virtual network to a tenant team.
      - Detect and root-cause an SLO violation (e.g. PFC storm, failing
        optic).
      - Replace a failed leaf.
- [ ] Functional requirements, numbered (`R-xxx`), grouped by area:
      model, rules, lifecycle, discovery/admission, device access,
      telemetry, UI, API, administration.
- [ ] Non-functional requirements: scale targets for v1 and long term,
      controller HA, headless operation (what the network does while
      the controller is down), security, upgradability, observability
      of Allium itself.
- [ ] Explicit non-goals for v1.
- [ ] Move the design statements now in requirements.md (e.g. ssh) into
      the design docs or proposals.

Decisions: P-01 (initial target and scope), P-03 (host/edge
participation), P-18 (multi-tenancy depth for v1).

### Phase 3 — Abstraction / Information Model

Goal: define what a network *is* to Allium. This is the core of the
project, and the API, UI and driver contract all follow from it.

- [ ] `model/overview.md`: the entity-relationship model. Covers
      device, interface (physical, sub, LAG), link, host, POP, virtual
      network, scope, rule, flow class, treatment, SLO, and driver
      capability. Show diagrams and example instances (a small
      spine-leaf).
- [ ] `model/scope-and-rules.md`: what a scope is (a set of devices,
      interfaces, POPs or networks? a predicate over them?), how rules
      attach to scopes, inheritance, precedence and conflict
      detection.
- [ ] `model/flows-and-treatment.md`: how flows are identified and
      classified, what treatment attributes exist, the default
      best-effort treatment, how SLOs are expressed, and how
      end-to-end SLOs decompose into per-hop treatments.
- [ ] `model/lifecycles.md`: state machines for devices
      (unknown → discovered → admitted → configured → operational →
      draining → retired), virtual networks and rules.
- [ ] A worked example that runs one use case through the whole model:
      intent → rules → per-device treatment → measurement.
- [ ] `model/capabilities.md`: the **Capability schema**, i.e. how a
      Driver declares which abstractions and treatments a Device
      supports, and how Allium handles partial support. background.md
      calls this schema critical to Allium's success, so give it its
      own document and review.

Decisions: P-04 (intent expression), P-05 (scope and rule semantics),
P-06 (flow identification), P-07 (treatment specification), P-08
(reconciliation authority), P-22 (Capability schema), P-25 (path
selection as Treatment).

### Phase 4 — Architecture

Goal: the main components and how they interact, written down before
any of them is designed in detail.

- [ ] Develop `architecture.md` into the architecture overview:
      - the Driver/Module split and how they connect (P-23);
      - a component diagram;
      - data flow from intent to device config, and from device
        telemetry back to SLO status;
      - Allium's role relative to distributed routing protocols
        (P-21).
- [ ] Management network and bootstrap
      (`design/management-network.md`, P-24).
- [ ] Device naming and identifiers, from "The naming of cats"
      (`design/naming.md`, P-13).
- [ ] Controller topology and resilience (`design/controller.md`).
- [ ] Southbound driver architecture and the definition of an
      "Allium Compliant Management Interface" (`design/southbound.md`).
- [ ] Northbound API style and the API-first principle
      (`design/northbound.md`).
- [ ] Choice of implementation language and major dependencies.

Decisions: P-09 (controller distribution), P-10 (state store /
platform), P-11 (southbound protocol), P-12 (driver packaging), P-14
(config deployment semantics), P-16 (northbound API), P-17
(implementation language), P-21 (role relative to routing protocols),
P-23 (Driver/Module integration), P-24 (management network and
bootstrap).

### Phase 5 — Subsystem Designs

Goal: each subsystem is designed in enough detail that it can be built.
These designs can run in parallel once Phase 4 has settled.

- [ ] Device identification and admission (`design/discovery-admission.md`).
- [ ] Discovery of devices, links and hosts.
- [ ] Reconciliation of intended, discovered and operational state
      (`design/reconciliation.md`).
- [ ] Rule compilation: intent → per-device treatment → driver
      operations.
- [ ] Telemetry, SLO evaluation, violation detection and alerting
      (`design/telemetry.md`).
- [ ] Security: users, RBAC/ReBAC, delegation, device credentials and
      trust (`design/security.md`).
- [ ] UI/UX: information architecture, topology view, heat map,
      virtual-network filter, rule application flow. Wireframes are
      enough at this stage.
- [ ] Test strategy: unit, simulation (e.g. SONiC VS, containerlab),
      hardware lab, conformance tests for drivers.

Decisions: P-13 (device identity and admission), P-15 (telemetry and
SLO measurement), P-18 (authorization model).

### Phase 6 — Risk-Reduction Prototypes (overlaps Phases 3–5)

Small, time-boxed spikes that answer design questions. Their output is
a short write-up, not production code.

- [ ] **SONiC southbound spike:** read state from and push config to
      SONiC VS in containerlab over gNMI and over the REST/config_db
      paths. Compare coverage and ergonomics. (Informs P-11.)
- [ ] **Compilation spike:** a toy compiler that turns a two-level
      scoped rule set into per-device QoS/ACL config for a 4-switch
      topology. (Informs P-05, P-07.)
- [ ] **Composition check spike:** given per-device treatment
      functions along a path, check a simple end-to-end property such
      as lossless class preserved or max queueing hops. (Informs the
      formal model in Phase 1/3.)
- [ ] **Telemetry spike:** stream queue and PFC counters from SONiC VS
      into a TSDB and compute a simple SLO indicator. (Informs P-15.)

### Phase 7 — MVP Implementation

Start only after the Phase 4 gate. Suggested MVP slice:

1. A single controller instance, with a clean seam left for later HA.
2. SONiC driver: inventory, interfaces, LLDP discovery, basic L3
   underlay, QoS/treatment classes.
3. Device admission (manual approval) and lifecycle state machine.
4. Intended topology plus reconciliation against discovered topology,
   with drift reporting.
5. Rules with two-level scope (fabric, virtual network), compiled to
   device config, with dry-run/diff.
6. Telemetry for interface and queue counters, a basic SLO indicator,
   and alerts.
7. Northbound API and CLI. The UI comes after the API is stable.
8. A containerlab-based end-to-end test environment in CI.

### Phase 8 — Beyond MVP

- Controller HA and distribution.
- Host/NIC driver (RoCE/DCQCN settings, host-side telemetry).
- Graphical UI: topology, heat map, virtual-network view.
- Hierarchical administration and delegation.
- Root-cause analysis for SLO violations.
- Additional device drivers and a driver conformance suite.

## Near-Term Next Steps

1. Review this plan and `proposals.md`, and reorder or cut as needed.
2. Decide the high-leverage proposals first: **P-01** (scope), **P-03**
   (edge participation), **P-04** (intent expression), **P-05**
   (scope semantics) and **P-21** (role relative to routing
   protocols). Most other work depends on them. **P-22** (Capability
   schema) and **P-23** (Driver/Module integration) come next.
3. Do Phase 0 and finish Phase 1: formalize the partial-map model and
   write the prior-art section. Both are cheap and unblock everything
   else.
4. Draft personas and use cases (Phase 2). They are the best tool for
   settling the scope proposals.
5. Start the SONiC southbound spike early. It is the main external
   unknown, and it runs independently of the doc work.
