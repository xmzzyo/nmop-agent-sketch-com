---
title: "Operational Requirements for Network State Exchange in Agent-Assisted Network Operations"
abbrev: "agent-state-req"
category: info

docname: draft-cui-nmop-agent-sketch-com-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Network Management Operations"
keyword:
 - Agent
 - Network State Exchange
 - Sketch
 - Network Management
venue:
  group: "Network Management Operations"
  type: "Working Group"
  mail: "nmop@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/nmop/"
  github: "xmzzyo/nmop-agent-sketch-com"
  latest: "https://xmzzyo.github.io/nmop-agent-sketch-com/draft-cui-nmop-agent-sketch-com.html"

author:
- role:
  ins: Y. Cui
  name: Yong Cui
  org: Tsinghua University
  street:
  city:
  region: Beijing
  code: 100084
  country: China
  phone:
  email: cuiyong@tsinghua.edu.cn
  uri: http://www.cuiyong.net/
- role:
  ins: M. Xing
  name: Mingzhe Xing
  org: Zhongguancun Laboratory
  street:
  city:
  region: Beijing
  code: 100094
  country: China
  phone:
  email: xingmz@zgclab.edu.cn
- role:
  ins: L. Zhang
  name: Lei Zhang
  org: Zhongguancun Laboratory
  street:
  city:
  region: Beijing
  code: 100094
  country: China
  phone:
  email: zhanglei@zgclab.edu.cn

normative:
  RFC2119:
  RFC8174:

informative:
  RFC6241:
    title: Network Configuration Protocol (NETCONF)
    author:
    - name: R. Enns
    - name: M. Bjorklund
    - name: J. Schoenwaelder
    - name: A. Bierman
  RFC7011:
    title: Specification of the IP Flow Information Export (IPFIX) Protocol for the Exchange of Flow Information
    author:
    - name: B. Claise
    - name: B. Trammell
    - name: P. Aitken
  RFC7950:
  RFC8040:
    title: RESTCONF Protocol
    author:
    - name: A. Bierman
    - name: M. Bjorklund
    - name: K. Watsen
  RFC8955:
    title: Dissemination of Flow Specification Rules
    author:
    - name: C. Loibl
    - name: S. Hares
    - name: R. Raszuk
    - name: D. McPherson
    - name: M. Bacher
  GNMI:
    title: gRPC Network Management Interface (gNMI)
    target: https://datatracker.ietf.org/doc/html/draft-openconfig-rtgwg-gnmi-spec
    date: 2018
  I-D.ietf-nmop-network-anomaly-architecture:
  I-D.ietf-nmop-network-anomaly-lifecycle:
  I-D.ietf-nmop-network-anomaly-semantics:
  I-D.ietf-nmop-network-incident-yang:
  I-D.ietf-nmop-yang-message-broker-integration:
  I-D.ietf-nmop-message-broker-telemetry-message:
  CORMODE:
    title: An Improved Data Stream Summary The Count-Min Sketch and its Applications
    author:
    - name: G. Cormode
    - name: S. Muthukrishnan
  SIPHASH:
    title: SipHash A Fast Short-Input PRF
    author:
    - name: J.-P. Aumasson
    - name: D. J. Bernstein

--- abstract

This document describes operational requirements for exchanging network state in agent-assisted network operations. In this document, an agent-assisted system is any automation or decision-support system that consumes operational state to support tasks such as anomaly triage, incident correlation, configuration verification, traffic engineering analysis, or attack mitigation support. Such a system may use a large language model (LLM), a rule engine, a statistical model, or conventional software.

The document focuses on operational problems created by high-volume telemetry, cross-domain state sharing, privacy constraints, approximate state summaries, and auditability of state used by automated workflows. It identifies requirements for compact, scoped, mergeable, bounded-error, incrementally synchronized, and auditable network state artifacts. The document complements existing NMOP work on anomaly detection, incident management, and YANG Push/message broker integration. It does not define a new network management protocol, a new agent communication protocol, or a wire format.

--- middle

# Introduction

The operational complexity of modern networks has grown substantially. Networks now span multiple autonomous systems (ASes), administrative domains, and technology layers. Network management tasks such as anomaly triage, incident correlation, fault localization, configuration consistency checking, and traffic engineering analysis require timely collection, synthesis, and interpretation of large amounts of network state.

Operators increasingly use automation and decision-support systems to reduce the time required to diagnose and respond to operational events. Some deployments may include agent-assisted components, including LLM-based agents, to help correlate alerts, summarize evidence, generate hypotheses, or prepare recommendations for human review. These components do not remove the need for existing management systems, telemetry pipelines, operator policy, or human accountability.

Agent-assisted operations introduce a practical state exchange problem. An analysis component may need to compare flow behavior across many routers, estimate the number of affected sources, identify configuration drift, or correlate symptoms across domains. Supplying raw telemetry to each component is often impractical because of data volume, privacy constraints, management-plane limits, and the latency of downstream analysis.

This document therefore focuses on requirements for network state artifacts consumed by agent-assisted or automated workflows. These artifacts can be generated downstream of existing telemetry mechanisms, including NETCONF {{RFC6241}}, RESTCONF {{RFC8040}}, YANG data models {{RFC7950}}, IPFIX {{RFC7011}}, gNMI {{GNMI}}, YANG Push/message broker pipelines {{I-D.ietf-nmop-yang-message-broker-integration}}, and telemetry message schemas {{I-D.ietf-nmop-message-broker-telemetry-message}}.

## Scope

This document is scoped to the exchange of network state consumed by agent-assisted operational workflows. In this document, "network state" includes telemetry-derived measurements, traffic summaries, topology-related observations, configuration-derived summaries, incident evidence, and other operational facts used to support analysis or recommendations.

The requirements apply to systems in which state may be exchanged between devices, collectors, controllers, domain-level automation components, incident management systems, and agent-assisted analysis components. The requirements are independent of whether the consuming component is implemented using an LLM, a rule engine, a statistical model, or a conventional application.

## Non-Goals

This document does not define a new network management protocol.

This document does not update NETCONF, RESTCONF, IPFIX, CoAP, YANG Push, gNMI, or other existing management and telemetry protocols.

This document does not standardize bindings for any agent communication protocol. Such protocols may be used by particular agent-assisted systems, but the requirements in this document do not depend on them.

This document does not specify autonomous mitigation behavior. High-impact operational actions, such as configuration changes, filtering rules, route policy changes, or rollback operations, remain subject to the authorization, validation, approval, and audit procedures of the deployed network management environment.

This document discusses sketch-based summaries as a candidate technique. It does not require the use of sketches in all deployments and does not define a wire format for sketch exchange.

# Conventions and Terminology

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

Agent:
: A software entity that assists a network management workflow by collecting context, correlating evidence, generating recommendations, or coordinating with other components. An agent may be driven by an LLM, a rule engine, a statistical model, or a combination of methods.

State Artifact:
: A structured representation of network state exchanged between systems. A state artifact can contain raw state, derived state, or a compact summary, together with metadata describing its scope and provenance.

Sketch:
: A probabilistic data structure that provides a compact, bounded-error summary of a multiset or set of network observations. Examples include Count-Min Sketch, HyperLogLog, DDSketch, MinHash, and Bloom Filter.

Sketch Merge:
: The operation of combining two Sketch structures of the same type into a single structure whose estimates reflect the union of the underlying observation sets.

# Problem Statement

Agent-assisted operational workflows may require broad network context across many devices and time windows. For example, incident triage may need recent interface counters, flow records, routing changes, topology information, device health indicators, and configuration differences. Sending raw telemetry to each analysis component can create excessive storage, transport, and processing overhead.

Operational workflows often need answers to specific questions rather than full raw records. Examples include:

- Which source prefixes are likely to be heavy hitters during an attack?
- How many distinct sources are observed across a domain?
- Which links show latency distributions that differ from a baseline?
- Which devices have configuration sets that are inconsistent with their peers?
- Which flows are likely affected by a reported incident?

Raw telemetry can answer these questions, but it may be unnecessarily expensive to exchange. Conversely, a summary that is too lossy, lacks error metadata, or cannot be audited can mislead an automated workflow. The challenge is to represent state at the right granularity for the operational question.

Some incidents cross administrative boundaries. Examples include distributed denial-of-service attacks, inter-domain reachability failures, and multi-provider service incidents. In such cases, operators may need to share selected state with peers or with a coordinating system. However, raw flow records, customer identifiers, topology details, and configuration fragments may be sensitive or subject to policy restrictions.

Compact or approximate state summaries can reduce data movement, but they introduce uncertainty. If an agent-assisted workflow consumes approximate state without understanding error bounds, time coverage, source scope, or freshness, it may generate incorrect conclusions or overconfident recommendations.

Existing telemetry and management mechanisms provide important capabilities, including schema-driven configuration and state access, streaming telemetry, flow export, and message-broker integration. However, deployments that introduce agent-assisted analysis still need guidance on what properties exchanged state artifacts should have so that they are compact, mergeable, bounded in error, privacy-aware, and auditable.

# Requirements

This section defines initial operational requirements for network state exchange in agent-assisted network operations. The list is intended as a starting point for discussion.

## Compact State Representation (REQ-1)

Solutions SHOULD support network state representations that are substantially smaller than the raw telemetry, logs, flow records, or configuration data from which they are derived, when the operational query does not require full-fidelity raw data.

The compact representation MUST preserve enough information to answer the intended operational query within the accuracy, freshness, and confidence requirements of the workflow.

## Query and Scope Metadata (REQ-2)

An exchanged state artifact MUST identify the operational query or query class that it is intended to support.

An exchanged state artifact MUST identify its scope, including the source device set or domain, observation time interval, collection method, sampling policy if any, and relevant aggregation parameters.

An exchanged state artifact SHOULD include provenance metadata sufficient for an operator or an incident management system to trace the artifact back to the telemetry source, collector, or generation process. This is aligned with the provenance needs described by telemetry message work in NMOP {{I-D.ietf-nmop-message-broker-telemetry-message}}.

## Bounded and Reportable Error (REQ-3)

If a state artifact is approximate, the artifact MUST include the parameters needed to interpret its error behavior.

For probabilistic summaries, the artifact MUST report the applicable error parameters, such as relative error, false-positive probability, confidence level, or other algorithm-specific bounds.

Consumers of approximate state SHOULD treat the error metadata as part of the input to their reasoning and SHOULD expose uncertainty in any generated recommendation, incident report, or operator-facing explanation.

## Mergeability Across Devices and Domains (REQ-4)

Solutions SHOULD support merging of state artifacts from multiple devices, collectors, or administrative domains when the operational query requires aggregate visibility.

A merge operation MUST preserve or update the scope metadata of the resulting artifact so that the combined device set, domain set, and time interval are explicit.

When merging approximate artifacts, the resulting artifact MUST include updated error metadata or indicate that the error behavior is unknown.

## Freshness and Time Alignment (REQ-5)

An exchanged state artifact MUST include timestamp information sufficient to determine its freshness.

When a workflow combines artifacts from multiple sources, the system SHOULD make the observation windows visible to the consumer so that time skew and stale inputs can be detected.

## Incremental Synchronization (REQ-6)

Solutions SHOULD support incremental updates when only a subset of the represented state changes between synchronization points.

Solutions MUST provide a way to detect stale, missing, or inconsistent updates when incremental synchronization is used.

Solutions MUST support full resynchronization when incremental updates are incomplete or when the receiver cannot reconstruct the current state.

## Privacy-Preserving Exchange (REQ-7)

Solutions SHOULD support cross-domain state exchange without requiring disclosure of raw flow records, customer identifiers, full topology details, or other sensitive operational data when aggregate state is sufficient.

Solutions MUST make clear whether a state artifact may still leak sensitive information through keys, labels, repeated queries, low-cardinality sets, or correlations with external data.

Operators SHOULD be able to apply policy controls to determine which state artifacts may be shared, with whom, and at what granularity.

## Auditability and Reproducibility (REQ-8)

State artifacts used by agent-assisted workflows SHOULD be logged or referenced in a way that allows later audit of the evidence used by the workflow.

The audit record SHOULD include artifact identifiers, generation parameters, source scope, time interval, software or model version where applicable, and consumer identity.

When an operator-facing recommendation is generated from approximate state, the recommendation SHOULD identify the input artifacts and their uncertainty metadata.

## Interoperability with Existing Management Systems (REQ-9)

Solutions SHOULD integrate with existing network management and telemetry systems rather than requiring a parallel data collection infrastructure.

Solutions SHOULD be able to consume state derived from existing mechanisms such as NETCONF {{RFC6241}}, RESTCONF {{RFC8040}}, YANG-modeled data {{RFC7950}}, IPFIX {{RFC7011}}, message brokers, time-series databases, and controller APIs.

Compact state artifacts generated downstream of YANG Push/message broker pipelines SHOULD preserve useful source and schema metadata from those pipelines {{I-D.ietf-nmop-yang-message-broker-integration}}.

## Separation from Operational Action (REQ-10)

State exchange mechanisms MUST NOT by themselves imply authorization to perform operational actions.

High-impact actions that are influenced by exchanged state, such as filtering, routing changes, configuration updates, or rollback operations, MUST remain subject to the authorization, validation, approval, and audit procedures of the deployment.

Approximate state SHOULD NOT be the sole basis for unattended high-impact action unless the operator has explicitly defined the applicable policy, risk threshold, validation process, and rollback procedure.

# Relationship to NMOP Work

This document is intended to complement existing NMOP work, rather than replace it.

The network anomaly architecture {{I-D.ietf-nmop-network-anomaly-architecture}}, anomaly lifecycle {{I-D.ietf-nmop-network-anomaly-lifecycle}}, and anomaly semantics {{I-D.ietf-nmop-network-anomaly-semantics}} describe how operational evidence can be collected, annotated, validated, and used in anomaly detection workflows. The requirements in this document focus on the properties of state artifacts that may feed such workflows.

The network incident YANG model {{I-D.ietf-nmop-network-incident-yang}} provides a structure for incident management. Compact state artifacts can be referenced as incident evidence or diagnostic inputs.

The YANG Push/message broker integration work {{I-D.ietf-nmop-yang-message-broker-integration}} and telemetry message model {{I-D.ietf-nmop-message-broker-telemetry-message}} address telemetry transport, schema, and provenance. Compact state artifacts can be generated downstream of such pipelines and should preserve relevant provenance and scope metadata.

# Example Candidate: Sketch-Based State Summaries

Sketch structures are one candidate representation for compact state artifacts exchanged by agent-assisted workflows. Rather than transmitting raw flow records, routing tables, or interface statistics for every query, a deployment can exchange Sketch summaries that answer specific questions about network state with bounded or measurable error.

A Sketch is not a detection tool or anomaly detector. It is a candidate summary artifact that may be consumed by operational systems. Its usefulness depends on the operational query, selected parameters, acceptable error bounds, and audit requirements.

The appropriate Sketch type depends on the nature of the network state being represented and the queries agents need to answer:

| Operational Task          | Query Type                                    | Candidate Summary      | Key Property Used                                    |
| ------------------------- | --------------------------------------------- | ---------------------- | ---------------------------------------------------- |
| Flow rate analysis        | "What is the traffic rate from prefix X?"     | Count-Min Sketch (CMS) | Frequency estimation with epsilon-delta bounds       |
| Source diversity analysis | "How many unique source IPs are there?"       | HyperLogLog (HLL)      | Cardinality estimation, cross-domain mergeable       |
| Latency / jitter analysis | "What is the p99 latency on path P?"          | DDSketch               | Quantile estimation with relative error bounds       |
| Configuration consistency | "Is device A's config consistent with peers?" | MinHash                | Set similarity estimation (Jaccard index)            |
| Affected flow marking     | "Is flow F affected by fault X?"              | Bloom Filter           | Set membership with configurable false positive rate |

If Sketches are used, their artifacts need to carry scope, freshness, provenance, and error metadata as described in Section 4.

# Initial Use Cases

This section lists initial use cases. More detailed operator use cases are expected in future revisions.

DDoS evidence exchange:
: A domain can publish a compact heavy-hitter or cardinality summary as evidence for an incident workflow. A peer or coordinating system can use the artifact to assess whether an attack appears distributed without requiring raw flow records. Mitigation, if any, is performed through existing mechanisms such as FlowSpec {{RFC8955}} or local filtering procedures.

Multi-domain fault localization:
: Domains can exchange latency or loss distribution summaries for aligned time windows. A coordinating workflow can identify where behavior deviates from baseline and request additional evidence from the relevant domain.

Configuration consistency verification:
: A collector can publish compact configuration-set similarity artifacts. An audit workflow can identify devices that appear inconsistent and hand them to existing configuration management systems for operator review.

Traffic engineering analysis:
: A traffic engineering workflow can consume summarized traffic matrix and latency artifacts to prepare recommendations. Any approved routing or policy change is applied through existing operational procedures.

# Security Considerations

State artifacts exchanged over untrusted networks need authentication, integrity protection, and confidentiality appropriate to the sensitivity of the represented state.

Credential management should bind a consuming component to an operational role, administrative domain, and permitted state scope.

Sketch structures or other compact state artifacts could be tampered with to influence agent-assisted analysis. Transport security can prevent eavesdropping and impersonation, but deployments should also consider artifact-level integrity protection where artifacts are stored, forwarded, or consumed asynchronously.

An adversary with write access to a summary generation function could manipulate summaries to cause incorrect analysis or recommendations. Defenses include using keyed hash functions such as SipHash {{SIPHASH}} for Sketch index computation, cross-validating estimates from multiple independent sources, and monitoring for statistically anomalous summary patterns.

State generation and exchange functions can be targets for denial-of-service attacks. Implementations should enforce rate limits, quotas, back pressure, and admission control for artifact generation and retrieval.

Approximate state summaries can also leak information through repeated queries, low-cardinality sets, or correlation with external data. Sharing policies need to account for such leakage risks.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
