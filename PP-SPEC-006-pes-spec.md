# Proof of Efficacy Score (PES) Specification

**Document ID:** PP-SPEC-006  
**Version:** 1.0  
**Status:** Published  
**License:** CC BY 4.0  
**Maintained by:** Proof Economy™ Standards Alliance (PESA)  
**Repository:** https://github.com/proofprotocol  
**Published:** 2026-07-12  

---

## Abstract

This specification defines the Proof of Efficacy Score (PES): the numeric metric expressing the containment performance of a security tool or autonomous agent as measured under the Proof Protocol™. It defines the formula, the case classification taxonomy, the denominator logic, and the requirements for a PES score to be considered a valid proof metric rather than a marketing number.

PES is the quantitative layer of the Proof Economy™. It answers the question: of all the adversarial actions that could have been contained, how many were?

---

## Status of This Document

This document is a published specification of the Proof Protocol™. It is released under the Creative Commons Attribution 4.0 International License (CC BY 4.0). You are free to share, implement, and build on it for any purpose including commercially, provided attribution is given to Nebulonium, Inc. / HACKERverse and the Proof Economy™ Standards Alliance (PESA).

---

## Table of Contents

1. [Motivation](#1-motivation)
2. [Terminology](#2-terminology)
3. [Case Classification Taxonomy](#3-case-classification-taxonomy)
4. [The PES Formula](#4-the-pes-formula)
5. [Denominator Logic](#5-denominator-logic)
6. [What Is Not a Valid PES Score](#6-what-is-not-a-valid-pes-score)
7. [Supplementary Metrics](#7-supplementary-metrics)
8. [PES in Proof Records](#8-pes-in-proof-records)
9. [Reference Implementation](#9-reference-implementation)
10. [Conformance](#10-conformance)
11. [Relationship to Other Proof Protocol™ Specifications](#11-relationship-to-other-proof-protocol-specifications)
12. [IANA Considerations](#12-iana-considerations)
13. [References](#13-references)
14. [Authors](#14-authors)

---

## 1. Motivation

Security scores are everywhere. Detection rates, containment percentages, block rates, efficacy claims - vendors publish these numbers constantly. Buyers have no way to verify them.

The problem is structural. A score without a declared denominator is not a metric. A detection rate without a definition of what counts as detected is not measurable. A containment percentage that excludes inconvenient cases from the denominator without declaring the exclusion is not honest.

PES exists to define what a valid efficacy score looks like - one that is formula-bound, denominator-declared, case-classified, and anchored to a tamper-evident proof chain. A PES score is not a claim. It is a calculation over a declared set of classified execution events, verifiable by anyone with access to the proof chain.

---

## 2. Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119.

**Proof of Efficacy Score (PES):** A pair of scores expressing containment and detection performance. Both MUST be published together. Neither may be published alone.

**Containment Score:** `BLOCKED / (BLOCKED + OBSERVED + MISSED) × 100`. The proportion of in-scope adversarial cases whose effect the SUT prevented.

**Detection Score:** `DETECTED / (BLOCKED + OBSERVED + MISSED) × 100`, where DETECTED is the count of in-scope cases with `detected: true`. The proportion of in-scope adversarial cases the SUT identified, whether or not it contained them.

**Case:** A single adversarial action, TTP execution, or test event presented to the System Under Test during a proof run.

**Classification:** The disposition assigned to a case following execution. One of: BLOCKED, MISSED, OBSERVED, IRRELEVANT. Defined in Section 3.

**In-Scope Case:** A case classified as BLOCKED, MISSED, or OBSERVED. These cases enter the denominator of both scores.

**Out-of-Scope Case:** A case classified as IRRELEVANT. The only classification excluded from the denominator, and only where the exclusion was pre-declared.

**Denominator:** The sum of BLOCKED, MISSED, and OBSERVED cases. Identical for both scores.

**Detected:** A boolean recorded per case. True for BLOCKED and OBSERVED by definition. Either value for MISSED. Omitted for IRRELEVANT.

**System Under Test (SUT):** The security tool, platform, or agent whose efficacy is being scored.

**Proof Run:** A complete execution of test cases against a SUT under the Proof Protocol™, producing a proof chain with classified case records.

**ProofRegister™:** The append-only public registry where completed proof bundles including PES scores are anchored.

---

## 3. Case Classification Taxonomy

Every case in a proof run MUST be assigned exactly one classification. Classification MUST occur at the time the case result is recorded in the proof chain. Post-hoc reclassification is not permitted.

### 3.1 BLOCKED

The SUT detected and contained the adversarial action. The action did not achieve its intended effect.

**Criteria:**
- The SUT identified the action as adversarial
- The SUT took a containment action (block, quarantine, terminate, reject, or equivalent)
- The containment action prevented the adversarial effect from materializing
- All three criteria must be met for a BLOCKED classification

**Enters PES denominator:** Yes  
**Enters PES numerator:** Yes

### 3.2 MISSED

The adversarial action was presented to the SUT and was not contained. The action achieved its intended effect, in whole or in part.

**Criteria:**
- The action was presented to the SUT within declared scope
- The SUT did not take a containment action, OR the containment action was insufficient to prevent the adversarial effect
- Either criterion is sufficient for a MISSED classification

**Enters PES denominator:** Yes  
**Enters PES numerator:** No

### 3.3 OBSERVED

The adversarial action was presented to the SUT and was detected or logged but not contained. The SUT produced a signal (alert, log entry, telemetry event) indicating awareness of the action without blocking it.

**Rationale for exclusion from denominator:** OBSERVED cases represent detection without containment. Including them in the denominator would allow a SUT that detects everything but blocks nothing to claim a non-zero PES score. Detection and containment are distinct capabilities. PES measures containment only.

**Note:** OBSERVED cases SHOULD be reported as a supplementary metric (Detection Rate). See Section 7.

**Enters PES denominator:** No  
**Enters PES numerator:** No

### 3.4 IRRELEVANT

The case was not applicable to the SUT in its declared configuration or scope. The case was presented but the SUT had no declared capability or responsibility to act on it.

**Criteria:**
- The case falls outside the declared scope of the proof run, OR
- The SUT's declared capability set explicitly excludes this case type, AND
- The exclusion was declared in the proof run parameters before execution began

**Important:** IRRELEVANT classification requires pre-execution scope declaration. A case cannot be reclassified as IRRELEVANT after execution because the result was inconvenient. Post-hoc IRRELEVANT assignments are classification fraud.

**Enters PES denominator:** No  
**Enters PES numerator:** No

### 3.5 Classification Decision Tree

```
Was the case within declared scope?
├── NO → IRRELEVANT (only if scope exclusion was pre-declared)
└── YES → Did the SUT take a containment action?
           ├── YES → Did containment prevent the adversarial effect?
           │          ├── YES → BLOCKED         (detected: true)
           │          └── NO  → MISSED          (detected: true)
           └── NO  → Did the SUT produce any detection signal?
                      ├── YES → OBSERVED        (detected: true)
                      └── NO  → MISSED          (detected: false)
```

---

## 4. The PES Formula

```
Denominator = BLOCKED + OBSERVED + MISSED

Containment = (BLOCKED / Denominator) × 100
Detection   = (DETECTED / Denominator) × 100
```

Where DETECTED is the count of in-scope cases with `detected: true`.

Both scores are expressed as percentages rounded to one decimal place.

IRRELEVANT cases are excluded from the denominator. No other classification may be excluded.

Both scores MUST be published together with the full classification breakdown and the detected count.

**Example:**

| Classification | Count | Detected |
|----------------|-------|----------|
| BLOCKED | 120 | 120 |
| OBSERVED | 42 | 42 |
| MISSED | 2 | 0 |
| IRRELEVANT | 3 | n/a |
| **Total Cases** | **167** | |

```
Denominator = 120 + 42 + 2 = 164
Containment = (120 / 164) × 100 = 73.2%
Detection   = (162 / 164) × 100 = 98.8%
```

IRRELEVANT (3) does not enter either calculation.

### 4.1 Edge Cases

**Zero in-scope cases:** If a run produces zero BLOCKED, zero OBSERVED, and zero MISSED cases, both scores are undefined and MUST be reported as N/A with a declaration that no in-scope cases were recorded. Scores of 100% MUST NOT be reported for a run with zero in-scope cases.

**Containment above detection:** Structurally impossible. Every BLOCKED case is detected by definition, so Containment can never exceed Detection. A published pair where it does indicates a classification error and the run MUST be rejected.

---

## 5. Denominator Logic

The denominator is `Blocked + Missed`. This is the total count of cases where the SUT had both the opportunity and the declared responsibility to act.

### 5.1 Why OBSERVED Enters the Denominator

An OBSERVED case is one the SUT saw and did not stop. The adversarial effect succeeded.

Earlier versions of this specification excluded OBSERVED from the denominator on the reasoning that including it would let vendors inflate scores by alerting rather than blocking. That reasoning was inverted. Excluding OBSERVED is what rewards alerting over blocking: a case the SUT declines to contain disappears from the measurement entirely, so a product that detects everything and stops nothing scores identically to one that stops everything.

OBSERVED is a containment failure and enters the containment denominator. It is a detection success and enters the detection numerator. The two-score model records both facts without either concealing the other.

### 5.2 Why IRRELEVANT Is Excluded

IRRELEVANT cases were outside the declared scope. Including them would allow vendors to pad the denominator with cases they were never expected to handle, artificially lowering the MISSED count relative to the total.

### 5.3 Denominator Declaration Requirement

A PES score MUST be accompanied by the following declared values:

```json
{
  "pes_score": "<percentage to one decimal place>",
  "pes_numerator": "<count of BLOCKED cases>",
  "pes_denominator": "<count of BLOCKED + MISSED cases>",
  "total_cases": "<total count of all cases>",
  "blocked_count": "<integer>",
  "missed_count": "<integer>",
  "observed_count": "<integer>",
  "irrelevant_count": "<integer>",
  "proof_chain_reference": "<ProofRegister™ campaign ID>",
  "pes_calculated_utc": "<ISO 8601>"
}
```

A PES score published without these accompanying values is not a valid PES score under this specification.

---

## 6. What Is Not a Valid PES Score

### 6.1 A Score Without a Denominator

Publishing "99% containment" without declaring the total case count, the classification breakdown, and the denominator composition is not a PES score. It is an unverifiable claim.

### 6.2 A Score Where OBSERVED Cases Enter the Denominator

Any scoring methodology that includes OBSERVED cases in the containment denominator produces a metric that conflates detection with containment. This is not PES. It may be a valid detection metric under a different name, but it MUST NOT be presented as a containment score.

### 6.3 A Score Derived From a Self-Reported Log

PES requires a proof chain. A percentage calculated from a vendor's own log file, without pre-execution commitment, chain continuity, and independent anchoring, is not a PES score under this specification.

### 6.4 A Post-Hoc Score

A score calculated after execution from reconstructed case records is not a valid PES score. PES must be calculated from a proof chain where each case was classified at the time of recording, under a pre-execution commitment.

### 6.5 A Score With Undeclared Exclusions

If cases were excluded from the denominator for any reason other than IRRELEVANT classification pre-declared in the proof run scope, the score is invalid. Silent exclusion of inconvenient cases is not scope management - it is score manipulation.

---

## 7. Supplementary Metrics

PES measures containment only. The following supplementary metrics SHOULD be reported alongside PES to provide a complete picture of SUT performance. They are not components of PES and MUST NOT be conflated with it.

### 7.1 Detection Rate

```
Detection Rate = ((Blocked + Observed) / (Blocked + Missed + Observed)) × 100
```

Measures the SUT's ability to identify adversarial actions regardless of whether it contained them. IRRELEVANT cases are excluded.

### 7.2 False Positive Rate (FPR)

```
FPR = False Positives / (False Positives + True Negatives) × 100
```

Measures the rate at which the SUT incorrectly classified benign actions as adversarial. Requires a benign case set in addition to the adversarial case set.

### 7.3 Coverage Rate

```
Coverage Rate = Applicable Cases / Total Cases × 100
```

Measures what fraction of total cases were applicable to the SUT. A low coverage rate warrants scrutiny of scope declarations.

### 7.4 Supplementary Metric Declaration

Supplementary metrics MUST be clearly labeled and MUST NOT be presented in a way that could be confused with PES. A vendor reporting Detection Rate alongside PES MUST label each metric distinctly.

---

## 8. PES in Proof Records

### 8.1 Case Record Classification Field

Every proof chain case record MUST include a `classification` field set to one of: `BLOCKED`, `MISSED`, `OBSERVED`, `IRRELEVANT`, and a `detected` boolean.

`detected` MUST be true for BLOCKED and OBSERVED. It MAY be either value for MISSED. It is omitted for IRRELEVANT.

```json
{
  "record_id": "<integer>",
  "record_type": "TTP_EXECUTION",
  "ttp_id": "<TTP identifier>",
  "classification": "BLOCKED | MISSED | OBSERVED | IRRELEVANT",
  "detected": true,
  "classification_rationale": "<brief explanation>",
  "containment_action": "<description if BLOCKED or MISSED>",
  "detection_signal": "<description if detected is true>",
  "irrelevant_scope_reference": "<pre-declared scope exclusion if IRRELEVANT>",
  "content_hash": "<SHA-256>",
  "previous_record_hash": "<SHA-256>",
  "chain_hash": "<SHA-256>",
  "record_timestamp_utc": "<ISO 8601>"
}
```

### 8.2 PES Summary Record

A proof chain MUST include a PES Summary Record as its penultimate record before the final anchor record.

```json
{
  "record_type": "PES_SUMMARY",
  "pes_score": "<percentage>",
  "pes_numerator": "<integer>",
  "pes_denominator": "<integer>",
  "total_cases": "<integer>",
  "blocked_count": "<integer>",
  "missed_count": "<integer>",
  "observed_count": "<integer>",
  "irrelevant_count": "<integer>",
  "detection_rate": "<percentage>",
  "false_positive_rate": "<percentage or null if not measured>",
  "coverage_rate": "<percentage>",
  "pes_calculated_utc": "<ISO 8601>",
  "content_hash": "<SHA-256>",
  "previous_record_hash": "<SHA-256>",
  "chain_hash": "<SHA-256>"
}
```

---

## 9. Reference Implementation

The first PES score computed under this specification was produced during the certification run of Pipelock v3.0.0 by Josh Waldrep, an agentic egress firewall, conducted under the Proof Protocol™ and anchored to ProofRegister™.

| Field | Value |
|-------|-------|
| Product | Pipelock v3.0.0 |
| Campaign ID | PR-2026-00028 |
| Anchor Block | Block 29 |
| NIST Beacon Pulse | 1852788 |
| Total Cases | 167 |
| In-Scope Cases | 164 |
| BLOCKED | 120 |
| OBSERVED | 42 |
| MISSED | 2 |
| IRRELEVANT | 3 |
| Detected | 162 |
| **Containment Score** | **73.2%** |
| **Detection Score** | **98.8%** |
| Child TTP Records | 117 |
| Certification | ProofStamp™ |

Query the registry: [proofregister.com](https://proofregister.com)

---

## 10. Conformance

An implementation conforms to this specification if:

1. It classifies every case as exactly one of BLOCKED, MISSED, OBSERVED, or IRRELEVANT
2. It records a `detected` boolean for every in-scope case
3. It applies both formulas as defined in Section 4 without modification
4. It excludes only IRRELEVANT cases from the denominator
5. It publishes the Containment Score and the Detection Score together, never one alone
6. It does not reclassify cases after execution
7. It declares the full case count breakdown and detected count alongside every published score
8. It records classification and detection in the proof chain at the time of case execution
9. It includes a PES Summary Record in every proof chain
8. It does not present Detection Rate or any other supplementary metric as PES

---

## 11. Relationship to Other Proof Protocol™ Specifications

| Document | Relationship |
|----------|-------------|
| Proof Protocol™ Specification (PP-SPEC-001) | Core protocol. PES Summary Record is a chain record type under PP-SPEC-001. |
| Proof Validity Specification (PP-SPEC-002) | A score without denominator declaration fails PP-SPEC-002 Section 4.7. PES is the valid metric PP-SPEC-002 requires. |
| ProofBundle™ Format Specification (PP-SPEC-003) | PES Summary Record and case classification records are carried in the ProofBundle™. |
| ProofRegister™ API Specification (PP-SPEC-004) | PES scores are queryable via the ProofRegister™ API by campaign ID. |
| Agent-to-Agent Proof Protocol™ (PP-SPEC-007) | PES scores computed during agentic proof runs follow this specification. OBSERVED classification is particularly relevant for agentic detection-without-containment outcomes. |
| ProofStamp™ Certification Criteria (PP-SPEC-009) | ProofStamp™ certification requires a minimum PES threshold as defined in PP-SPEC-009. |

---

## 12. IANA Considerations

This document has no IANA considerations.

---

## 13. References

- RFC 2119 - Key words for use in RFCs: https://www.rfc-editor.org/rfc/rfc2119
- Proof Protocol™ Specification (PP-SPEC-001): https://github.com/proofprotocol
- Proof Validity Specification (PP-SPEC-002): https://github.com/proofprotocol
- MITRE ATT&CK Framework: https://attack.mitre.org
- Creative Commons CC BY 4.0: https://creativecommons.org/licenses/by/4.0/

---

## 14. Authors

Proof Economy™ Standards Alliance (PESA)  
https://proofeconomy.foundation  
contact@proofeconomy.foundation  

*This specification is maintained by PESA. Governance of this specification follows the PESA practitioner-led model. Vendors may contribute but do not govern.*

---

*Copyright 2026 Nebulonium, Inc. dba HACKERverse. Licensed under CC BY 4.0.*  
*ProofStamp™ is a certification mark of Nebulonium, Inc.*
