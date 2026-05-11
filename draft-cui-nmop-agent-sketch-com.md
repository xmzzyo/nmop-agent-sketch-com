---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "Distributed Sketch and Agent Communication Framework for Network Operations"
abbrev: "agent-sketch-com"
category: info

docname: draft-cui-nmop-agent-sketch-com-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: Operations and Management Area
workgroup: NMOP
keyword:
 - Agent
 - Skecth
 - Network Management
venue:
  group: Network Management
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
- role:  # remove if not true
  ins: Y. Cui
  name: Yong Cui
  org: Tsinghua University
  street:
  city:
  region: Beijing # not always available
  code: 100084
  country: China # use TLD (except UK) or country name
  phone:
  email: cuiyong@tsinghua.edu.cn
  uri: http://www.cuiyong.net/
- role: # remove if not true
  ins: M. Xing
  name: Mingzhe Xing
  org: Zhongguancun Laboratory
  street:
  city:
  region: Beijing # not always available
  code: 100094
  country: China # use TLD (except UK) or country name
  phone:
  email: xingmz@zgclab.edu.cn
- role: # remove if not true
  ins: L. Zhang
  name: Lei Zhang
  org: Zhongguancun Laboratory
  street:
  city:
  region: Beijing # not always available
  code: 100094
  country: China # use TLD (except UK) or country name
  phone:
  email: zhanglei@zgclab.edu.cn

normative:
  [RFC2119]:

  [RFC7252]:

  [RFC7641]:

  [RFC7950]:

  [RFC7959]:

  [RFC8174]:

  [RFC8949]:

  [RFC9147]:


informative:
  RFC6241:
    title: Network Configuration Protocol (NETCONF)
    author:
    - name: R. Enns
    - name: M. Bjorklund
    - name: J. Schoenwaelder
    - name: A. Bierman
    date: 2011-06

  RFC7011:
    title: Specification of the IP Flow Information Export (IPFIX) Protocol for the Exchange of Flow Information
    author:
    - name: B. Claise
    - name: B. Trammell
    - name: P. Aitken
    date: 2013-09

  RFC8040:
    title: RESTCONF Protocol
    author:
    - name: A. Bierman
    - name: M. Bjorklund
    - name: K. Watsen
    date: 2017-01

  RFC8955:
    title: Dissemination of Flow Specification Rules
    author:
    - name: C. Loibl
    - name: S. Hares
    - name: R. Raszuk
    - name: D. McPherson
    - name: M. Bacher
    date: 2020-12

  RFC9254:
    title: Encoding of Data Modeled with YANG in the Concise Binary Object Representation (CBOR)
    author:
    - name: M. Veillette
    - name: I. Petrov
    - name: A. Pelov
    - name: C. Bormann
    - name: M. Richardson
    date: 2022-07

  mcp:
    title: Model Context Protocol
    target: https://modelcontextprotocol.io
    date: 2024

  a2a:
    title: Agent-to-Agent (A2A) Protocol
    target: https://google.github.io/A2A
    date: 2025

  gNMI:
    title: gRPC Network Management Interface (gNMI)
    target: https://datatracker.ietf.org/doc/html/draft-openconfig-rtgwg-gnmi-spec
    date: 2018

  CORMODE:
    title: An Improved Data Stream Summary: The Count-Min Sketch and its Applications
    author:
    - name: G. Cormode
    - name: S. Muthukrishnan
    date: 2005

  SIPHASH:
    title: SipHash: A Fast Short-Input PRF
    author:
    - name: J.-P. Aumasson
    - name: D. J. Bernstein
    date: 2012
...

--- abstract

This document describes a framework for efficient and reliable communication among AI-driven agents and between agents and network devices in the context of network operations (NetOps). As large language model (LLM)-based agents are increasingly deployed to automate network management tasks — including fault localization, configuration verification, traffic engineering, and attack mitigation — they must exchange large volumes of network state information across multiple administrative domains. Existing protocols are not designed to jointly satisfy the reliability requirements of operational commands and the efficiency requirements of network state dissemination at scale.

This document motivates the need for a new communication framework, defines requirements, and proposes an architecture that combines the Constrained Application Protocol (CoAP) for reliable message delivery with distributed probabilistic data structures (Sketch) for compact, mergeable network state representation. Bindings between CoAP and emerging agent protocols (MCP and A2A) are outlined. Representative use cases, including DDoS detection and mitigation, are described to validate the applicability of the framework.


--- middle

# Introduction

###  Context and Motivation

The operational complexity of modern networks has grown substantially. Networks now span multiple autonomous systems (ASes), administrative domains, and technology layers. Network management tasks — such as detecting and mitigating distributed denial-of-service (DDoS) attacks, localizing faults across domains, verifying configuration consistency, and optimizing traffic engineering — require the timely collection, synthesis, and reasoning over large amounts of network state.

Traditional approaches to network operations relied on human operators and rule-based automation. These approaches do not scale to the demands of large, dynamic, multi-domain networks. The widespread availability of high-quality large language models (LLMs) in the 2020s opened a new paradigm: AI-driven network operations (AI-driven NetOps), in which autonomous LLM-based agents perform complex reasoning tasks — root cause analysis, multi-step remediation planning, policy synthesis — that previously required significant human expertise. The IETF NMOP working group has been chartered to address emerging challenges in network management and operations, including the integration of AI and automation into the management plane. This document is a contribution to that effort.

A fundamental architectural insight is that a single LLM agent cannot maintain complete, real-time visibility over a large multi-domain network. The scale of telemetry data, the diversity of device types, and administrative separation between domains each impose hard limits on what any single agent can observe or control. A **multi-agent architecture** is therefore necessary: Orchestration Agents maintain a global view and coordinate responses across domains; Domain Agents aggregate state from devices within their domain; Device Agents (or device-side CoAP servers) maintain local state and execute instructions. This three-tier structure allows each level to operate with appropriate granularity, eliminating the information overload that would result from a flat, fully-connected agent topology.

###  The Communication Problem

Deploying cooperating agents in a production network introduces a fundamental communication challenge with two conflicting pressures.

The first is a **reliability requirement**. Network operations involve consequential actions: changing routing policies, applying access control lists, rate-limiting traffic, rolling back configurations. These actions must be executed with confirmation. An agent that issues a mitigation command to a border router and receives no confirmation cannot know whether the network is protected. Silent delivery failures are operationally dangerous. Commands must be acknowledged, retransmission must be idempotent, and failures must be reported.

The second is an **efficiency requirement**. The primary information currency between agents is network state — link utilization, flow statistics, routing tables, interface health, configuration parameters — and the volume of this data is enormous. A single edge router may generate millions of flow records per minute; a domain of hundreds of routers generates billions. Agents do not need raw data: they need actionable summaries — answers to questions such as "Which source prefix is sending the most traffic?" or "How many unique source IPs are observed across this domain?" Moreover, in multi-operator or multi-AS scenarios, raw flow records cannot be shared across administrative boundaries due to privacy, legal, and competitive constraints.

Existing protocols do not simultaneously address both requirements. NETCONF/YANG [RFC6241] and gNMI [GNMI] provide reliable, schema-driven management but produce verbose, full-fidelity output not suitable for agent-to-agent state exchange. IPFIX [RFC7011] provides efficient flow export but offers no reliability guarantees or agent-interaction semantics. The Model Context Protocol (MCP) [MCP] and Agent-to-Agent (A2A) protocol [A2A] provide the right agent-native semantics — tool invocation, task delegation, artifact exchange — but are currently defined over HTTP/SSE, which is ill-suited to constrained network device management planes, and define no mechanism for compressing the network state that agents must exchange.

| Protocol           | Reliable Delivery  | Efficient State  | Agent-Native               | Assessment                                    |
| ------------------ | ------------------ | ---------------- | -------------------------- | --------------------------------------------- |
| NETCONF/YANG       | Yes                | No (full XML)    | No                         | Too verbose; no agent semantics               |
| gNMI/gRPC          | Yes                | Partial          | No                         | No summary layer; heavy stack                 |
| IPFIX/NetFlow      | No                 | Partial          | No                         | Export only; no agent interaction             |
| MCP (HTTP)         | Partial            | No               | Yes                        | No transport guarantees; no compression       |
| A2A (HTTP/SSE)     | Partial            | No               | Yes                        | SSE not suitable for device management planes |
| **This framework** | **Yes (CoAP CON)** | **Yes (Sketch)** | **Yes (MCP/A2A bindings)** | **Addresses both requirements**               |

###  Proposed Approach: CoAP and Sketch

This document proposes a framework that resolves the reliability-efficiency tension through a **two-layer communication design**, with each layer addressing one dimension of the problem and the two layers combining cleanly.

**The Reliable Layer** is based on the Constrained Application Protocol (CoAP) [RFC7252]. CoAP operates over UDP and provides a reliable subset of HTTP semantics with a compact 4-byte binary header. Its Confirmable (CON) message type implements acknowledged delivery with exponential-backoff retransmission, directly satisfying the reliability requirement for operational commands. CoAP's Non-confirmable (NON) messages and Observe extension [RFC7641] provide loss-tolerant push notifications for high-frequency telemetry streams. DTLS [RFC9147] provides mutual authentication and encryption. CoAP is already widely implemented on network equipment and is the basis of existing IETF management standards such as COMI [RFC9254], giving the framework both the right transport properties and an established deployment footprint.

**The Efficiency Layer** is based on distributed probabilistic data structures — collectively referred to in this document as *Sketch*. Sketches provide compact, fixed-size representations of streaming network observations with provable bounded-error guarantees. A Count-Min Sketch summarizing per-flow traffic rates across a domain may occupy a few hundred kilobytes, compared to gigabytes of raw flow records. A HyperLogLog estimating the number of unique source IPs occupies 64 bytes regardless of how many distinct addresses are observed. Critically, Sketches support a **merge operation**: structures from multiple devices or domains can be combined into a structure representing the union of all observations, without access to the underlying raw data. This mergeability property enables cross-domain state aggregation and cross-domain sharing without exposing privacy-sensitive information.

The two layers are orthogonal and complementary: **CoAP governs how messages are delivered; Sketch governs what they contain.** Neither alone is sufficient — CoAP without Sketch would transmit raw telemetry and fail on efficiency and privacy; Sketch without CoAP would have no mechanism for reliable command delivery. Together, they allow each component to be evolved independently while providing a clean interface that both the network management and AI agent communities can implement.

Beyond CoAP and Sketch, the framework defines normative **bindings between CoAP and MCP/A2A**. MCP and A2A represent an emerging consensus on how AI agents communicate. Without defined bindings, each deployment builds its own translation layer, leading to fragmentation. This document defines those bindings (Section 8) to enable the NMOP working group to standardize a single, interoperable interface.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

**Agent:** A software entity capable of autonomous reasoning and action in a network management context. An agent may be driven by a large language model (LLM), a rule engine, or a combination of both.

**Orchestration Agent:** A high-level agent responsible for decomposing complex network management tasks, dispatching sub-tasks to Domain Agents, and synthesizing results into operational decisions. Typically LLM-driven.

**Domain Agent:** An agent responsible for a specific administrative domain (e.g., an autonomous system or a geographic region). It collects and summarizes network state from devices within its domain and cooperates with other Domain Agents and the Orchestration Agent.

**Device Agent:** A lightweight agent co-located with or embedded in a network device (router, switch). It manages local Sketch structures and exposes them via a CoAP server interface.

**Sketch:** A probabilistic data structure that provides a compact, bounded-error summary of a multiset or set of network observations. Examples include Count-Min Sketch, HyperLogLog, DDSketch, MinHash, and Bloom Filter.

**Sketch Node:** A network device or software component that maintains one or more Sketch structures updated from the local data plane (e.g., via P4, eBPF, or software sampling).

**Sketch Merge:** The operation of combining two Sketch structures of the same type into a single structure whose estimates reflect the union of the underlying observation sets.

**XOR-Delta:** An incremental transmission scheme in which only the changed cells of a Sketch array are transmitted between synchronization points, computed as the bitwise XOR of the current and baseline Sketch arrays.

**CoAP:** The Constrained Application Protocol [RFC7252], a lightweight RESTful protocol operating over UDP with optional reliability via Confirmable (CON) messages.

**MCP:** The Model Context Protocol [MCP], an open protocol that standardizes how applications provide context, tools, and resources to LLM-based agents.

**A2A:** The Agent-to-Agent protocol [A2A], a protocol for task-level communication and coordination between autonomous agents.

**CON message:** A CoAP Confirmable message that requires an acknowledgment (ACK) from the recipient. Used for reliable delivery.

**NON message:** A CoAP Non-confirmable message sent without requiring an acknowledgment. Used for high-frequency, loss-tolerant data streams.

**Observe:** A CoAP extension [RFC7641] that allows a client to register interest in a resource and receive notifications when the resource changes.


## Problem Statement

###  The Reliability Requirement

Network operations involve consequential actions: changing routing policies, applying ACLs, rate-limiting traffic, rolling back configurations. An agent that issues a mitigation command to a border router and does not receive confirmation of execution cannot know whether the network is protected. Silent failures — commands lost in transit — are operationally dangerous.

The reliability requirement has several dimensions:

- **Delivery guarantee:** Operational commands MUST be delivered to the target device and acknowledged.
- **Idempotency:** Retransmitted commands MUST NOT cause duplicate or inconsistent state changes on the device.
- **Ordering:** Related commands (e.g., a sequence of configuration steps) MUST be executed in the correct order.
- **Failure notification:** When a command cannot be delivered after retransmission, the issuing agent MUST be notified.

Existing approaches that rely on UDP-based telemetry streams without acknowledgment do not satisfy this requirement. TCP-based protocols satisfy it but introduce head-of-line blocking and connection overhead that is problematic for constrained devices and lossy management-plane paths.

###  The Efficiency Requirement

Network state is voluminous. Transmitting raw telemetry between agents at operational timescales is impractical for several reasons:

- **Volume:** A single domain may generate terabytes of raw telemetry per day. Agents cannot buffer or transmit this at the latency required for real-time operations.
- **Cross-domain privacy:** In multi-operator or multi-AS scenarios, raw flow records cannot be shared across administrative boundaries due to legal, regulatory, and competitive constraints.
- **Inference latency:** LLM agents reasoning over gigabytes of raw input incur unacceptable latency. Compact, structured summaries are required.
- **Management plane bandwidth:** The management plane of network devices is deliberately rate-limited to protect the control plane. High-volume telemetry export over this plane is not feasible.

The efficiency requirement demands a state representation that is compact, supports cross-domain sharing without exposing raw data, and can be incrementally updated to avoid full retransmission on every synchronization cycle.

###  The Gap in Existing Solutions

No existing protocol simultaneously satisfies the reliability requirement for operational commands, the efficiency requirement for network state dissemination, and the agent-native communication semantics required for LLM-driven NetOps. This three-way gap is the core problem this framework addresses.

The reliability-efficiency tension is not resolvable by adjusting parameters of any single existing protocol — it requires a deliberate two-layer design. The agent-native gap requires new bindings between emerging agent protocols (MCP, A2A) and network device interfaces. Both design choices are developed in Sections 5 through 8.

---

## Requirements

This section defines the requirements that the framework is designed to satisfy. Requirements are stated using the key words defined in Section 2.

###  Reliable Command Delivery (REQ-1)

The framework MUST provide a mechanism for delivering operational commands from an agent to a target device or agent with guaranteed delivery and acknowledgment.

The framework MUST support retransmission of unacknowledged commands with configurable backoff.

The framework MUST support idempotent command execution, such that a retransmitted command does not cause duplicate state changes on the target.

The framework MUST notify the sending agent when a command cannot be delivered after the maximum number of retransmission attempts.

###  Efficient Network State Representation (REQ-2)

The framework MUST support a compact representation of network state that can be exchanged between agents with substantially lower bandwidth than raw telemetry data.

The compact representation MUST provide provable, configurable error bounds (ε, δ) on the accuracy of estimates derived from it.

The compact representation MUST support merging of instances from multiple sources to produce a combined representation without access to the underlying raw data.

###  Incremental State Synchronization (REQ-3)

The framework MUST support incremental (delta) transmission of network state updates, such that only changes since the last synchronization point are transmitted.

The incremental transmission mechanism MUST support fallback to full-state transmission when the delta exceeds a configurable threshold or when a gap in the update sequence is detected.

###  Agent Protocol Integration (REQ-4)

The framework MUST define normative bindings between the agent communication primitives of MCP and A2A and the CoAP message types and resource model used by the framework.

The bindings MUST cover at minimum: tool invocation (MCP tools/call), resource subscription (MCP resources/subscribe), task submission (A2A tasks/send), and task status subscription (A2A tasks/subscribe).

###  Quantifiable Accuracy (REQ-5)

The framework MUST ensure that the error bounds (ε, δ) of Sketch estimates are communicated alongside the estimates themselves, so that agents can incorporate uncertainty into their reasoning and decision-making.

The framework SHOULD define how error bounds propagate through Sketch merge operations across multiple domains or devices.

###  Cross-Domain Privacy (REQ-6)

The compact state representation used by the framework MUST NOT require the transmission of raw flow records, raw IP addresses, or other privacy-sensitive data to satisfy cross-domain state sharing requirements.

The representation MUST allow agents in different administrative domains to derive useful aggregate estimates without exposing the underlying observations.

###  Incremental Deployability (REQ-7)

The framework SHOULD be deployable on existing network infrastructure without requiring hardware upgrades.

The framework SHOULD define a software-based implementation path (e.g., using Linux eBPF or user-space sampling) as a fallback to hardware-accelerated implementations (e.g., P4-based data plane Sketch updates), with documented performance trade-offs.

###  Interoperability with Existing Management Infrastructure (REQ-8)

The framework SHOULD be interoperable with existing network management infrastructure, including YANG data models [RFC7950] and NETCONF [RFC6241].

The framework SHOULD define a YANG module for Sketch node configuration and state, allowing Sketch nodes to be managed via existing NETCONF/RESTCONF tooling.

---

## Framework Architecture

###  Overview

The framework defines a three-tier agent architecture connected by a two-layer communication stack:

```
  ┌──────────────────────────────────────────────┐
  │         Orchestration Agent (LLM)            │
  │  Global reasoning · Task dispatch · Decision │
  └───────────────┬──────────────────────────────┘
                  │ A2A over CoAP
        ┌─────────┼─────────┐
        ▼         ▼         ▼
  ┌──────────┐ ┌──────────┐ ...
  │  Domain  │ │  Domain  │
  │  Agent   │ │  Agent   │
  └────┬─────┘ └────┬─────┘
       │ CoAP       │ CoAP
  ┌────▼──────────────▼────┐
  │  Network Devices       │
  │  (Sketch Nodes)        │
  └────────────────────────┘
```

- **Reliable Layer (CoAP):** Carries operational commands, task coordination messages, and large Sketch payloads with guaranteed delivery.
- **Efficiency Layer (Sketch):** Provides the data representation in all network state exchanges. Sketch structures are generated at Sketch Nodes, transmitted via CoAP to Domain Agents, merged at the domain level, and aggregated at the orchestration level.

###  Agent Roles and Responsibilities

**Orchestration Agent:**
- Receives NetOps task requests from operators or automated systems.
- Decomposes tasks into sub-tasks and delegates them to Domain Agents via A2A task messages.
- Aggregates Sketch summaries from multiple domains to derive global network state estimates.
- Makes operational decisions based on Sketch-derived estimates and LLM reasoning.
- Issues operational commands to Domain Agents or Sketch Nodes via CoAP CON messages.

**Domain Agent:**
- Subscribes to Sketch updates from all Sketch Nodes within its domain via CoAP Observe.
- Maintains a domain-level merged Sketch representing the aggregate state of its domain.
- Responds to Sketch sharing requests from the Orchestration Agent or peer Domain Agents.
- Executes sub-tasks assigned by the Orchestration Agent.

**Device Agent / Sketch Node:**
- Maintains one or more Sketch structures updated from the local data plane.
- Exposes Sketch resources via a CoAP server at well-known resource paths.
- Pushes incremental Sketch updates to subscribed Domain Agents via CoAP Observe NON messages.
- Receives and executes operational commands delivered via CoAP CON messages.

###  Communication Relationships

| Relationship                       | Protocol      | Primary Use                                                 |
| ---------------------------------- | ------------- | ----------------------------------------------------------- |
| Orchestration Agent ↔ Domain Agent | A2A over CoAP | Task delegation, Sketch aggregation, decision dissemination |
| Domain Agent ↔ Domain Agent        | A2A over CoAP | Cross-domain Sketch sharing, peer coordination              |
| Domain Agent ↔ Sketch Node         | CoAP (direct) | Sketch subscription, command delivery, status reporting     |

---

## CoAP-Based Reliable Communication

###  CoAP Profile for This Framework

This framework uses CoAP [RFC7252] as the transport substrate for all agent-to-device and agent-to-agent communication. The following features are used:

- **Confirmable (CON) messages** for operational commands, task messages, and large Sketch transfers, with ACK and exponential-backoff retransmission.
- **Non-confirmable (NON) messages** for high-frequency incremental Sketch updates via Observe. Loss-tolerant; a missed update is recovered at the next synchronization cycle.
- **Observe [RFC7641]** for Domain Agent subscriptions to device Sketch resources, receiving push notifications when Sketch state changes beyond a configured threshold.
- **Block-Wise Transfer [RFC7959]** for Sketch payloads exceeding the maximum CoAP message size.
- **CBOR encoding** (Content-Format 60) for all Sketch payloads [RFC8949], reducing payload size by 30–50% compared to JSON.

###  CoAP Resource Tree

Sketch Nodes and Device Agents MUST expose the following CoAP resource tree:

```
coap://<device>/
├── ops/sketch/
│   ├── ops/sketch/cms        (Count-Min Sketch)
│   ├── ops/sketch/hll        (HyperLogLog)
│   ├── ops/sketch/ddsketch   (DDSketch)
│   └── ops/sketch/minhash    (MinHash / Bloom Filter)
├── ops/agent/
│   ├── ops/agent/task        (Receive agent tasks via POST)
│   └── ops/agent/status      (Report agent status via GET/Observe)
└── ops/config/
    ├── ops/config/apply      (Apply configuration via CON POST)
    └── ops/config/rollback   (Rollback configuration via CON POST)
```

Devices MUST expose at minimum `ops/sketch/cms` and `ops/sketch/hll`.

###  Reliability Mechanisms

**Retransmission:** CON messages not acknowledged within ACK_TIMEOUT (default: 2 seconds per RFC 7252) are retransmitted with exponential backoff up to MAX_RETRANSMIT (default: 4) attempts. After MAX_RETRANSMIT failures, the sending agent MUST be notified of delivery failure.

**Idempotency:** Every CON command message MUST carry a unique Token (4 bytes) and a SequenceID in the payload. Receiving devices MUST maintain an idempotency cache keyed by (Token, SequenceID) with a configurable TTL (default: 300 seconds). Duplicate messages MUST return the cached response without re-executing the command.

**Gap detection:** Domain Agents MUST monitor Observe sequence numbers on subscribed Sketch resources. If a gap larger than a configurable threshold (default: 5 missed updates) is detected, the Domain Agent MUST issue a CON GET to retrieve the full current Sketch state and resynchronize the baseline.

---

## Sketch-Based Efficient State Representation

###  Sketch as Inter-Agent State Currency

In this framework, Sketch structures serve as the primary representation of network state exchanged between agents. Rather than transmitting raw flow records, routing tables, or interface statistics, agents exchange Sketch summaries — compact structures that answer specific queries about network state with bounded error.

A Sketch is not a detection tool or anomaly detector. It is a data representation format — the network state analog of a compressed file format, but one that supports meaningful queries and cross-domain merging. The intelligence — detection, reasoning, and decision-making — resides in the agents that query and interpret Sketch structures.

###  Sketch Type Selection

The appropriate Sketch type depends on the nature of the network state being represented and the queries agents need to answer:

| NetOps Task               | Query Type                                    | Recommended Sketch     | Key Property Used                                    |
| ------------------------- | --------------------------------------------- | ---------------------- | ---------------------------------------------------- |
| Flow rate analysis        | "What is the traffic rate from prefix X?"     | Count-Min Sketch (CMS) | Frequency estimation with ε-δ bounds                 |
| Source diversity analysis | "How many unique source IPs are there?"       | HyperLogLog (HLL)      | Cardinality estimation, cross-domain mergeable       |
| Latency / jitter analysis | "What is the p99 latency on path P?"          | DDSketch               | Quantile estimation with relative error bounds       |
| Configuration consistency | "Is device A's config consistent with peers?" | MinHash                | Set similarity estimation (Jaccard index)            |
| Affected flow marking     | "Is flow F affected by fault X?"              | Bloom Filter           | Set membership with configurable false positive rate |

Sketch parameters SHOULD be configured based on the expected observation cardinality and the desired accuracy level (ε, δ).

###  Incremental Transmission: XOR-Delta

Sketch structures are fixed-size arrays of counters or registers. Full retransmission at every synchronization interval is wasteful when only a small fraction of cells change. The XOR-Delta scheme provides efficient incremental updates:

1. The Sketch Node maintains the current array `S[t]` and the baseline `S[t0]` (state at last synchronization).
2. The delta is computed as `D = S[t] XOR S[t0]`.
3. Only the non-zero entries of D are transmitted as `(index, value)` pairs.
4. The receiving agent reconstructs the current Sketch: `S[t] = S[t0] XOR D`.
5. When the fraction of changed cells exceeds a threshold (default: 20%), or a gap in the Observe sequence is detected, full Sketch retransmission is triggered.

Under typical steady-state conditions, incremental deltas are expected to represent 1–5% of the full Sketch size, reducing management plane bandwidth consumption proportionally.

###  Error Bound Propagation

When Sketch structures from multiple sources are merged, the error bounds of the merged structure can be computed analytically for most Sketch types. For example, when two Count-Min Sketches with the same dimensions (w, d) and error parameters (ε, δ) are merged via element-wise maximum, the merged structure retains the same error parameters.

The framework requires that error bound parameters (ε, δ) be included in all Sketch messages so that receiving agents can propagate them correctly. Implementations SHOULD validate that Sketch structures being merged have compatible parameters before performing the merge operation.

---

## Agent Protocol Bindings

###  Binding Design Principles

The bindings defined in this section map MCP and A2A semantic primitives onto CoAP methods, message types, and resource paths. The guiding principles are:

- **Reliability follows semantics:** MCP/A2A primitives with operational consequences (state-modifying tool invocations, task submissions) MUST map to CoAP CON messages. Observational primitives (subscriptions, status updates) SHOULD map to CoAP NON with Observe.
- **Encoding efficiency:** All payloads SHOULD use CBOR encoding (Content-Format 60). JSON (Content-Format 50) MAY be used for diagnostic purposes.
- **Path stability:** CoAP resource paths defined in this framework MUST NOT change between protocol versions.

###  MCP-over-CoAP Binding

| MCP Primitive           | CoAP Method   | Message Type                  | Resource Path               |
| ----------------------- | ------------- | ----------------------------- | --------------------------- |
| `tools/list`            | GET           | CON                           | `/ops/mcp/tools`            |
| `tools/call`            | POST          | CON                           | `/ops/mcp/tools/call`       |
| `resources/read`        | GET           | CON                           | `/ops/mcp/resources/{name}` |
| `resources/subscribe`   | GET + Observe | CON (register) / NON (notify) | `/ops/sketch/{type}`        |
| `resources/unsubscribe` | RST           | —                             | CoAP RST to cancel Observe  |
| `prompts/get`           | GET           | CON                           | `/ops/mcp/prompts/{name}`   |

For `tools/call`, the MCP JSON-RPC 2.0 request body is carried as the CoAP payload. The CoAP Token field serves as the correlation identifier and MUST be unique per outstanding request.

###  A2A-over-CoAP Binding

| A2A Primitive        | CoAP Method   | Message Type                  | Notes                                               |
| -------------------- | ------------- | ----------------------------- | --------------------------------------------------- |
| `tasks/send` (sync)  | POST          | CON                           | Response carries task result directly               |
| `tasks/send` (async) | POST          | CON                           | Response is 2.31 Continue; task ID in Location-Path |
| `tasks/get`          | GET           | CON                           | Resource path: `/a2a/tasks/{task-id}`               |
| `tasks/cancel`       | DELETE        | CON                           | Resource path: `/a2a/tasks/{task-id}`               |
| `tasks/subscribe`    | GET + Observe | CON (register) / NON (notify) | Replaces HTTP SSE for task status streaming         |

For asynchronous tasks (the common case for complex NetOps tasks such as fault localization), the interaction proceeds as follows:

1. The initiating agent sends `tasks/send` via CON POST and receives a 2.31 Continue response with the task ID.
2. The initiating agent registers an Observe subscription on the task resource.
3. The executing agent sends NON Observe notifications as the task progresses; the final notification carries the task result artifacts.
4. The initiating agent cancels the Observe subscription by sending a CoAP RST.

###  AgentCard CoAP Extensions

Agents supporting this framework MUST include the following additional fields in their A2A AgentCard:

```json
{
  "coap_extensions": {
    "endpoint": "coap://<host>[:<port>]",
    "dtls_required": true,
    "observe_supported": true,
    "cbor_encoding": true,
    "max_payload_bytes": 1024,
    "block_transfer": true,
    "sketch_types": ["cms", "hll", "ddsketch", "minhash"]
  }
}
```

---

## Use Cases

###  Use Case 1: DDoS Detection and Mitigation

**Scenario:** A volumetric DDoS attack is directed at a destination prefix within AS-1. The attack traffic originates from a large botnet distributed across multiple ASes.

**Participating entities:** Border routers in AS-1 (Sketch Nodes), Domain Agent for AS-1, peer Domain Agents for AS-2 and AS-3, Orchestration Agent.

**Sketch usage:** Border routers maintain Count-Min Sketch structures updated by eBPF programs on the data plane, tracking per-source-prefix packet rates. Each Domain Agent maintains a HyperLogLog to estimate the cardinality of unique source IP addresses across its domain.

**Protocol flow:**

1. Domain Agent AS-1 receives CMS incremental updates (XOR-Delta, NON Observe) from border routers. It queries the merged CMS and detects that traffic from source prefix 203.0.113.0/24 has exceeded a configured threshold.

2. Domain Agent AS-1 sends an A2A `tasks/send` (DDOS_SUSPECT) message via CON POST to the Orchestration Agent, attaching its domain HLL (64 bytes) as a task artifact.

3. The Orchestration Agent sends A2A `SKETCH_SYNC_REQUEST` tasks to Domain Agents for AS-2 and AS-3, requesting their HLL Sketches.

4. The Orchestration Agent merges the three HLLs to estimate the total unique source IPs across all domains. A cardinality above 10^5 indicates a distributed botnet; below 100 suggests a single-source amplification attack requiring a different response.

5. The Orchestration Agent determines the appropriate mitigation action (e.g., FlowSpec [RFC8955] rate-limit rule) and delivers it to border routers in AS-1 via CON POST to `/ops/config/apply`. CON guarantees delivery; idempotency ensures retransmission does not cause duplicate ACL entries.

6. Routers acknowledge application (2.04 Changed). Domain Agent AS-1 continues pushing CMS deltas; the Orchestration Agent monitors whether traffic normalizes and issues a CON POST to remove the rule when the attack subsides.

**Requirements addressed:** REQ-1 (reliable mitigation delivery), REQ-2 (HLL/CMS compact representation), REQ-6 (cross-domain sharing without raw IP exposure).

###  Use Case 2: Multi-Domain Fault Localization

**Scenario:** Users in AS-1 report packet loss to a destination in AS-3. The fault may lie in any of the transit ASes.

**Protocol flow:** Domain Agents for each AS query their DDSketch structures tracking per-path latency and loss distributions, and share merged DDSketches via A2A tasks. The Orchestration Agent compares per-hop quantile estimates to identify the AS where latency or loss deviates from baseline, then queries the relevant Domain Agent for Bloom Filter data marking affected flows to narrow down the faulty link.

**Requirements addressed:** REQ-2, REQ-3, REQ-4 (async A2A task binding), REQ-6.

###  Use Case 3: Configuration Consistency Verification

**Scenario:** A network-wide audit is required to verify that all border routers are running consistent BGP policy configurations.

**Protocol flow:** The Orchestration Agent requests MinHash Sketches from all Domain Agents. Each Domain Agent computes MinHash structures representing the set of active configuration items on its devices. The Orchestration Agent computes pairwise Jaccard similarity estimates to identify devices whose configurations have diverged. Divergent devices are flagged, and corrective configurations are pushed via CON POST.

**Requirements addressed:** REQ-1, REQ-2, REQ-5 (MinHash similarity bounds).

###  Use Case 4: Traffic Engineering Optimization

**Scenario:** The Orchestration Agent needs to optimize inter-domain traffic routing based on current load and latency conditions.

**Protocol flow:** Domain Agents continuously aggregate per-flow CMS structures from their devices into domain-level traffic matrices. DDSketch structures capture latency distributions on inter-domain links. The Orchestration Agent periodically collects these structures via A2A tasks (using Block-Wise Transfer for large matrices), constructs a compressed global traffic matrix, and uses LLM reasoning to generate updated routing policy recommendations. Approved policies are pushed via CON POST.

**Requirements addressed:** REQ-1, REQ-2, REQ-3, REQ-7.

---

## Deployment Considerations

###  Incremental Deployment Path

The framework is designed for incremental deployment without simultaneous upgrades across all devices:

**Phase 1 — Software-based Sketch:** Sketch structures are maintained by user-space or eBPF programs on existing device CPUs. CoAP servers run as software daemons. No hardware changes required; deployable immediately on Linux-based equipment (REQ-7).

**Phase 2 — eBPF-accelerated Sketch:** eBPF XDP programs update Sketch structures at several million packets per second on commodity NICs, providing near-line-rate collection for high-traffic edge devices without programmable forwarding hardware.

**Phase 3 — P4-based hardware Sketch:** On P4-programmable forwarding hardware, Sketch updates occur at full line rate (100 Gbps and beyond) entirely within the data plane. P4 programs export Sketch deltas to the device's CoAP server process for transmission to Domain Agents.

###  YANG Model Integration

A companion document defines a YANG module `ietf-sketch-node` modeling Sketch type configuration, Sketch state (current values, timestamps, error bounds), CoAP server configuration, and subscription management. This allows operators to configure and monitor Sketch Nodes using existing NETCONF/RESTCONF tooling (REQ-8).

###  Sketch Parameter Selection Guidelines

- **Count-Min Sketch:** Set width `w = ceil(e / ε)` and depth `d = ceil(ln(1/δ))` where ε is the desired maximum relative frequency error and δ is the desired failure probability.
- **HyperLogLog:** Relative standard error ≈ 1.04 / √m where m = 2^b is the number of registers. For 2% error, use b = 12 (m = 4096), occupying 1.5 KB with 4-bit registers.
- **DDSketch:** The relative accuracy parameter α (default: 0.01) determines the maximum relative error on quantile estimates.

# Security Considerations


###  Authentication and Encryption

All CoAP communication MUST be protected by DTLS 1.3 [RFC9147] when operating over untrusted networks:

- **Certificate mode:** Used for agent-to-agent communication. Each agent presents an X.509 certificate whose Subject or SAN field identifies its CoAP URI. Certificates are issued by a management-plane PKI.
- **Pre-shared key (PSK) mode:** Used for agent-to-device communication where devices have limited computational resources. PSK values are provisioned out-of-band and can be rotated by the Orchestration Agent via CON POST.

###  Sketch Integrity

Sketch structures transmitted between agents and devices could be tampered with to influence agent decision-making. DTLS encryption prevents eavesdropping; DTLS authentication prevents impersonation. Additionally, each Sketch message SHOULD carry an HMAC-SHA256 integrity tag over the payload, keyed with a secret negotiated during the DTLS handshake.

###  Sketch Poisoning

An adversary with write access to a Sketch Node could manipulate Sketch structures to cause incorrect agent decisions. Defenses include:

- Using keyed hash functions (e.g., SipHash [SIPHASH]) for Sketch index computation, preventing predictable collision attacks.
- Cross-validating Sketch estimates from multiple independent Sketch Nodes before acting on them.
- Monitoring for statistically anomalous Sketch patterns (e.g., a single cell accounting for an implausibly large fraction of total counts).

### Agent Authorization

The framework RECOMMENDS an authorization model in which:

- Device Agents accept CON commands only from Domain Agents whose certificates identify them as authorized for the device's domain.
- Domain Agents accept Sketch sharing requests only from agents with certificates signed by a trusted management authority.
- Orchestration Agents are constrained to the operational scope defined in their certificates (e.g., AS number, administrative domain).

### Denial of Service

CoAP servers on network devices are resource-constrained and could be overwhelmed by floods of CON messages. Implementations SHOULD enforce per-source rate limits on incoming CON messages and SHOULD use CoAP's built-in congestion control mechanisms (ACK_TIMEOUT, NSTART).



# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
