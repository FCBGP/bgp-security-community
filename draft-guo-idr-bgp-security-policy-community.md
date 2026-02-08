---
title: "Origin-Authorized Routing Policy Signaling Using BGP Communities"
abbrev: "BGP Security Policy Community"
category: std

docname: draft-guo-idr-bgp-security-policy-community-latest
submissiontype: IETF # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Routing"
workgroup: "Inter-Domain Routing"
keyword:
  - BGP Community
  - Routing Security
venue:
  # group: "Inter-Domain Routing"
  # type: "Working Group"
  # mail: "idr@ietf.org"
  # arch: "https://mailarchive.ietf.org/arch/browse/idr/"
  github: "FCBGP/bgp-security-community"
  latest: "https://FCBGP.github.io/bgp-security-community/draft-guo-idr-bgp-security-community.html"

author:
  - name: Yangfei Guo
    org: Zhongguancun Laboratory
    email: guoyangfei@zgclab.edu.cn
  - fullname: Ke Xu
    org: Tsinghua University
    email: xuke@tsinghua.edu.cn
  - fullname: Xiaoliang Wang
    org: Capital Normal University
    email: wangxiaoliang0623@foxmail.com

normative:
  RFC1997: # BGP Communities Attribute
  RFC4271: # BGP protocol
  RFC4360: # BGP Extended Communities Attribute
  RFC8092: # BGP Large Communities Attribute

informative:
  RFC6480: # RPKI infrastructure
  RFC6482: # RPKI-ROA
  RFC8205: # BGPsec protocol
  RFC9234: # Role, OTC, Route Leak Prevention
  RFC9582: # RPKI-ROA-bis
  ASPA-Profile: I-D.ietf-sidrops-aspa-profile
  ASPA-Verification: I-D.ietf-sidrops-aspa-verification
  SIDROPS-STATE: I-D.ietf-sidrops-avoid-rpki-state-in-bgp
---

--- abstract

This document specifies a set of standardized BGP communities to signal inter-AS routing security policy intent. Current mechanisms like ROA {{RFC6482}} {{RFC9582}} and ASPA {{ASPA-Profile}} {{ASPA-Verification}} provide validation states, but leave "NotFound" or "Unknown" states ambiguous. This document defines transitive communities that allow an Origin AS to explicitly signal its security requirements, such as strict enforcement of ROA and ASPA, to downstream Autonomous Systems (AS). This mechanism provides downstream ASes with the explicit authorization to treat unvalidated routes as invalid, reducing the risk of accidental outages while improving hijack resilience. Unlike validation states, these communities communicate the origin's security expectations (e.g., ROA/ASPA strict enforcement) to all downstream ASes. The document emphasizes the transitive nature of these signals to ensure end-to-end security policy coordination. By enabling explicit signaling of security requirements, this mechanism allows downstream ASes to make informed decisions and enforce consistent policies, improving the overall security of inter-AS routing without risking accidental outages due to misinterpretation of validation states.

--- middle

# Introduction

Inter-domain routing security mechanisms intentionally separate validation from policy. While this separation improves robustness, it also creates persistent operational ambiguity.

Internet routing security relies on distributed validation mechanisms like RPKI {{RFC6480}}, ROA {{RFC6482}} {{RFC9582}}, and ASPA {{ASPA-Profile}} {{ASPA-Verification}}. However, there is a functional gap between "knowing a route's validity" and "knowing the origin's policy intent."

These security mechanisms are often locally enforced only. No consistent method exists for an AS to signal its security requirements for propagated prefixes. {{SIDROPS-STATE}} advises against carrying actual validation state in BGP. This leaves downstream ASes with ambiguity. For example, when a Transit AS observes an RPKI "NotFound" state, it cannot distinguish between an Origin AS that has not deployed RPKI and an Origin AS that has deployed RPKI but suffered a configuration error or a hijack attempt. Theses mechanisms do not allow an Origin AS to express:

- whether it expects strict validation to be applied;
- whether certain propagation behaviors are explicitly authorized;
- and whether "NotFound" or "Unknown" outcomes are operationally acceptable.

As a result, downstream networks frequently face situations where routing information is technically acceptable, yet operationally unexpected. In large-scale deployments, this ambiguity leads to:

- inconsistent treatment of identical prefixes;
- reliance on bilateral or undocumented conventions;
- and difficulty distinguishing misconfiguration from malicious behavior.

Existing uses of BGP communities partially address this gap, but lack standardized semantics and clear separation from validation state.

This document addresses the need for a minimal, standardized mechanism to express origin-authorized security policy intent, without redefining or exporting validation outcomes. It introduces BGP Security Policy Communities to bridge these gaps. These are signals that travel with the NLRI to inform every AS on the AS-PATH about the protection level applied to the prefix. By signaling security policy intent, an Origin AS can explicitly inform the network that its prefixes MUST always satisfy specific security criteria. For example, an Origin AS can signal that its prefixes MUST be rejected if they are not RPKI valid, even if the local RPKI check returns "NotFound". This allows downstream ASes to make informed decisions and enforce security policies consistently, without risking accidental outages due to misinterpretation of validation states.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Policy Signaling and Transitivity

## Transitive Property Requirement

All communities defined in this document MUST be transitive to ensure the security intent reaches the entire AS-PATH.

- Large Communities: This document will use large communities to carry the security policy sign of Origin AS. As per {{RFC8092}}, the Global Administrator field MUST contain the ASN of the Origin AS that set the policy.
- Propagation: Intermediate ASes SHOULD NOT strip these communities. If an AS modifies the AS-PATH (e.g., via AS-Prepend), it MUST maintain these communities.

## Policy Signaling Versus Validation State

This document makes a strict distinction between:

- validation state, which is derived locally from cryptographic or registry-based mechanisms; and
- policy intent, which reflects the Origin AS’s operational expectations.

The communities defined herein exclusively signal policy intent. They do not encode validation outcomes, confidence levels, or security posture of any AS other than the origin.

Downstream ASes MUST NOT interpret these communities as an indication that validation has been successfully performed, nor as a substitute for local validation.

This design explicitly aligns with the guidance in {{SIDROPS-STATE}}, and avoids propagating dynamic security state within BGP.

# Protocol Operations

## Architecture Overview

The mechanism defined in this document operates entirely within the BGP control plane and does not alter protocol message formats or path selection procedures.

An Origin AS attaches one or more security policy communities when originating a route. These communities are propagated unchanged unless explicitly removed by policy.

Intermediate and receiving ASes may choose to:

- ignore the communities;
- log or monitor their presence;
- correlate them with local validation results;
- and incorporate them into local policy decisions.

No mandatory processing behavior is defined, and no interoperability dependency is introduced.

## Origin AS Behavior

The Origin AS SHALL attach the appropriate Security Policy Community when advertising a prefix. It MUST ensure that its published RPKI ROAs and ASPA objects are consistent with the signaled community to avoid self-inflicted DoS. For example, an AS 65001 supporting strict RPKI-ROA filtering would attach: "Large Community: `65001:1000:1` (ROA-Strict)"

## Intermediate AS Behavior

Intermediate ASes SHOULD preserve these communities. If an Intermediate AS supports this specification, it MAY use these communities to automate its ingress/egress filtering policies. An AS receiving a route with these communities MUST process them as follows:

1. Validation: The AS MUST verify that the Global Administrator ASN in the Large Community matches the Rightmost (Origin) AS in the AS-PATH. If they do not match, the community MUST be ignored to prevent unauthorized policy signaling.
2. ROA-Strict Enforcement: If `ROA-Strict` is present and the local RPKI check returns `NotFound` or `Invalid`, the AS SHOULD treat the route as `Invalid` and apply its local filtering policy (typically dropping the route).
3. ASPA-Strict Enforcement: If `ASPA-Strict` is present and the ASPA validation state is `Unknown` or `Invalid`, the AS SHOULD reject the route.

## Applicability to Route Leak and Hijack Detection

Security policy communities may serve as additional context for routing analysis systems.

For example, a route that violates an origin-authorized export constraint, while also exhibiting abnormal AS path patterns, may be flagged as anomalous with higher confidence.

Such signals can reduce false positives in detection systems by providing operator-declared intent, without asserting correctness. This document does not define detection algorithms or mitigation procedures.

## Deployment Considerations

The proposed mechanism is compatible with existing routing policies and does not require changes to BGP implementations. Deployment of this mechanism is expected to be incremental and partial.

The proposed mechanism is intended for gradual deployment and interoperability with existing BGP technologies. Origin ASes may selectively signal policy intent for specific prefixes. In hybrid networks where both supporting and non-supporting ASes are present, policy communities will simply be ignored by legacy BGP speakers, providing backward compatibility. No coordination between ASes is required.

For environments supporting both Standard {{RFC1997}} and Large {{RFC8092}} Communities, implementations SHOULD attach both representations to maximize backward compatibility. Operators are encouraged to monitor for loss or modification of policy communities due to intermediate ASes that filter or rewrite BGP Community attributes so as to ensure policy expectations are properly signaled end-to-end.

# Security Considerations

## Authenticity

These communities provide no security guarantees and MUST NOT be used as a basis for accepting otherwise invalid routes.

While communities are transitive, they are not cryptographically signed (unlike BGPsec). An adversary could potentially attach or strip these communities. Therefore, these communities SHOULD be treated as "Policy Recommendations" unless combined with BGPsec or other path validation mechanisms. And these communities MUST NOT override a "Valid" RPKI state. They serve as a "Strictness Toggle" for states that are otherwise ambiguous.

## Mitigation of DoS Risks

By removing active "drop" commands, this document minimizes the risk of a malicious actor using these communities to trigger a network-wide outage. The mandatory check against the Origin AS in the AS-PATH ensures that only the legitimate owner of the prefix (or someone who has successfully hijacked the entire path) can signal the policy.

## Threat Model and Abuse Considerations

The communities defined in this document are not cryptographically protected. An adversary capable of hijacking a prefix may also attach, modify, or remove communities.

Therefore, these communities MUST NOT be treated as authoritative proof of security properties.

Potential abuse scenarios include:

- false signaling of strict security requirements;
- removal of policy signals during propagation;
- inconsistent signaling across multiple origin points.

These risks are consistent with existing uses of BGP communities and do not introduce new attack vectors. Operators SHOULD treat these signals as advisory inputs and correlate them with independently verifiable information.

# IANA Considerations

This document defines new BGP Community values for signaling security policy intent. IANA is requested to create a sub-registry "BGP Security Policy Action IDs" under the "BGP Large Communities" registry.

The format of these communities are `Global-Administrator:Action-ID:Parameter`. The Global Administrator MUST be the ASN of the Origin AS.


   Action ID |   Name      | Policy Intent Description
  ----------------------------------------------------------------------------------
    1000     |  ROA-Strict | Reject if RPKI state is Invalid OR NotFound.
    1001     | ASPA-Strict | Reject if ASPA validation state is Invalid or Unknown.

## Standard Community Mapping (Legacy)

For {{RFC1997}} support, the following values are assigned from the Well-Known range:

- `65535:1000` (ROA-Strict)
- `65535:1001` (ASPA-Strict)

--- back

# Parameter Field Extensibility

The "Parameter" field in the BGP Security Policy Community is explicitly designed for extensibility. Currently, a value of "1" conveys default strict enforcement for security actions (e.g., ROA-Strict or ASPA-Strict). Future assignments may introduce further parameters to support nuanced policy signaling, such as variant filtering levels, time-limited policies, or security requirements specific to more recent routing security enhancements.

Example encodings:

- `65001:1000:1` (ROA-Strict, default strict enforcement)
- `65001:1000:2` (ROA-Strict, but only for prefixes with maxLength constraints, a future possible assignment)
- `65001:1002:1` (hypothetical new Action-ID for a future policy, e.g., “AS-Cones Strict”)

The Action-ID and Parameter range is managed through IANA for orderly growth as the community adopts richer security policies.

# Acknowledgments
{:numbered="false"}

TBD.
