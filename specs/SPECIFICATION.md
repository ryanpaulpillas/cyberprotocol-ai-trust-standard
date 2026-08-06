# CyberProtocol AI Trust Standard: Technical Specification

**Version 1.0 (Draft, Initial Proposal)**
**Status:** Draft for public review. Data formats are proposed and may change before v1.0 is ratified.
**Steward:** One Planet One Earth Foundation Inc. (holder of UN ECOSOC Special Consultative Status since 2025)
**License:** Specification under CC BY 4.0. Reference code under Apache 2.0.

---

## Abstract

CyberProtocol defines an open, neutral, cryptographic framework for verifying the identity,
provenance, safety, and compliance of artificial intelligence and its outputs across
jurisdictions. This document specifies the data models, cryptographic suites, and verification
procedures that a conforming implementation uses. CyberProtocol introduces no new cryptography;
it profiles and unifies existing standards (W3C Decentralized Identifiers, W3C Verifiable
Credentials, W3C Data Integrity, and zero-knowledge proof systems) into one interoperable system.

---

## 1. Conformance and terminology

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED,
MAY, and OPTIONAL in this document are to be interpreted as described in RFC 2119 and RFC 8174
when, and only when, they appear in all capitals.

| Term | Definition |
|------|------------|
| **Agent** | An AI system that produces outputs under the control of an Operator. |
| **Operator** | The legal entity accountable for an Agent and its keys. |
| **Subject** | The output (text, file, or transaction) that a Seal certifies. |
| **Seal** | A signed attestation binding an Agent to a Subject at a point in time. |
| **Verifier** | Any party that validates a Seal using open tools and public key material. |
| **Resolver** | A service or library that returns the DID Document for a DID. |

A **conforming producer** MUST be able to issue Seals per Section 4. A **conforming verifier**
MUST implement the algorithm in Section 6. An implementation MUST declare which of the four
pillars (Section 3) it supports.

---

## 2. Architecture

CyberProtocol is organized as four layers that compose into a single verifiable claim:

```
        ┌─────────────────────────────────────────────┐
        │  Pillar 4  Cross-Border Verification         │  neutral, open validation
        ├─────────────────────────────────────────────┤
        │  Pillar 3  Safety & Risk Compliance          │  EU AI Act / NIST / ISO
        ├─────────────────────────────────────────────┤
        │  Pillar 2  Provenance & Output Certification │  CyberSeal + proof
        ├─────────────────────────────────────────────┤
        │  Pillar 1  AI & Human Identity               │  did:cyber / W3C VC
        └─────────────────────────────────────────────┘
```

A single Seal (Pillar 2) references an identity (Pillar 1), carries compliance metadata
(Pillar 3), and is expressed in a format any party can verify (Pillar 4).

---

## 3. Pillar 1: Identity

### 3.1 Human identity

Human and organizational identities MUST be expressed as W3C Verifiable Credentials.
CyberProtocol does not define new human-identity credentials; it consumes existing ones.

### 3.2 The `did:cyber` method

AI agents are identified by a Decentralized Identifier using the `cyber` method.

**Identifier syntax (ABNF):**

```abnf
cyber-did       = "did:cyber:" entity-type ":" method-id
entity-type     = "agent" / "org"
method-id       = multibase-key / registry-ref
multibase-key   = "z" 1*base58char        ; self-certifying, multibase base58-btc
registry-ref    = 1*idchar                ; anchored in a governance registry
```

A `method-id` of the form `z...` is a self-certifying public key (as in `did:key`), requiring
no external infrastructure. Implementations MAY instead anchor a `registry-ref` in a governance
registry for rotation and revocation.

### 3.3 DID Document

An agent's DID Document MUST bind at least one verification key to the agent's model, version,
and controlling operator.

```json
{
  "@context": ["https://www.w3.org/ns/did/v1", "https://cyberprotocol.io/ns/v1"],
  "id": "did:cyber:agent:z6Mkf5rGMoatrSj1f4Cyvu...",
  "controller": "did:web:example-lab.org",
  "model": { "name": "example-llm", "version": "3.1", "release": "2026-05-01" },
  "verificationMethod": [{
    "id": "did:cyber:agent:z6Mkf5rGMoatrSj1f4Cyvu...#sign-key-1",
    "type": "Multikey",
    "controller": "did:cyber:agent:z6Mkf5rGMoatrSj1f4Cyvu...",
    "publicKeyMultibase": "z6Mkf5rGMoatrSj1f4Cyvu..."
  }],
  "assertionMethod": ["did:cyber:agent:z6Mkf5rGMoatrSj1f4Cyvu...#sign-key-1"]
}
```

**Requirements:**

- `controller` MUST identify the Operator (a DID, commonly `did:web`).
- `model` MUST record `name` and `version`; `release` is RECOMMENDED.
- At least one `verificationMethod` MUST be present and referenced by `assertionMethod`.
- Key rotation and revocation MUST be expressible; a revoked key MUST NOT validate Seals
  created after its revocation time.

---

## 4. Pillar 2: The CyberSeal

A Seal is a JSON object (canonical form) that MAY also be serialized as CBOR. It certifies
one Subject.

### 4.1 Members

| Member | Req. | Type | Description |
|--------|:----:|------|-------------|
| `@context` | MUST | URI/array | MUST include `https://cyberprotocol.io/ns/v1`. |
| `type` | MUST | string | MUST be `"CyberSeal"`. |
| `issuer` | MUST | DID | The Agent that produced the Subject. |
| `created` | MUST | datetime | ISO 8601 (UTC) time of issuance. |
| `subject.digest` | MUST | string | Multihash of the Subject bytes (Section 4.2). |
| `config.hash` | SHOULD | string | Commitment to Agent configuration. |
| `compliance` | SHOULD | object | Compliance metadata (Section 5). |
| `expires` | MAY | datetime | Time after which the Seal is stale. |
| `proof` | MUST | object | A Data Integrity proof over the Seal (Section 4.3). |

### 4.2 Subject digest

The Subject digest MUST be a multihash string of the form `sha-256:<hex>` (or another
registered hash). Producers MUST compute the digest over the exact byte sequence of the
Subject. Verifiers MUST recompute and compare in constant time.

### 4.3 Proof and canonicalization

Seals MUST be signed with a W3C Data Integrity proof. The canonicalization and hashing follow
the declared `cryptosuite`. Conforming implementations MUST support at least one of:

| Cryptosuite | Curve / Scheme | Canonicalization |
|-------------|----------------|------------------|
| `ecdsa-jcs-2019` | ECDSA (P-256) | JSON Canonicalization Scheme (RFC 8785) |
| `eddsa-jcs-2022` | Ed25519 | JSON Canonicalization Scheme (RFC 8785) |

The `proof.verificationMethod` MUST reference a key in the issuer's DID Document, and
`proof.proofPurpose` MUST be `assertionMethod`.

### 4.4 Example

```json
{
  "@context": "https://cyberprotocol.io/ns/v1",
  "type": "CyberSeal",
  "issuer": "did:cyber:agent:z6Mkf5rGMoatrSj1f4Cyvu...",
  "created": "2026-08-05T09:30:00Z",
  "subject": { "digest": "sha-256:8f434346648f6b96df89dda901c5176b10a6d8396..." },
  "config": { "hash": "sha-256:2c26b46b68ffc68ff99b453c1d30413413422d70648..." },
  "compliance": { "euAiAct": "limited-risk", "nistRmf": "measured", "iso42001": "attested" },
  "proof": {
    "type": "DataIntegrityProof",
    "cryptosuite": "ecdsa-jcs-2019",
    "created": "2026-08-05T09:30:00Z",
    "verificationMethod": "did:cyber:agent:z6Mkf5rGMoatrSj1f4Cyvu...#sign-key-1",
    "proofPurpose": "assertionMethod",
    "proofValue": "z58DAdFfa9SkqZMVvx..."
  }
}
```

### 4.5 Zero-knowledge mode (informative)

Where Agent configuration is a trade secret, an Operator MAY replace `config.hash` and selected
`compliance` fields with a zero-knowledge proof that attests to a statement (for example, "this
Agent's configuration is a member of an approved set" or "risk category is limited-risk")
without revealing the underlying values. The ZK circuit and public inputs are out of scope for
v1.0 and will be profiled in a companion document.

---

## 5. Pillar 3: Compliance vocabulary

The `compliance` object maps a single Seal to multiple regulatory frameworks at once.

| Key | Proposed values | Maps to |
|-----|-----------------|---------|
| `euAiAct` | `prohibited`, `high-risk`, `limited-risk`, `minimal-risk` | EU AI Act risk categories |
| `nistRmf` | `governed`, `mapped`, `measured`, `managed` | NIST AI RMF functions |
| `iso42001` | `attested`, `not-attested` | ISO/IEC 42001 attestation |

Values MUST come from the enumerations above unless extended by a registered profile.
Absence of a key means the Operator makes no claim for that framework. A compliance claim is an
assertion by the Operator; it is not, by itself, third-party certification.

---

## 6. Pillar 4: Verification

A Verifier MUST perform the following steps and MUST treat any failure as an invalid Seal.

1. **Parse.** Reject a Seal whose `type` is not `CyberSeal` or that lacks a required member.
2. **Resolve.** Resolve `issuer` to its DID Document via a Resolver.
3. **Bind key.** Confirm `proof.verificationMethod` is present in the DID Document and is not
   revoked as of `created`.
4. **Verify proof.** Verify the Data Integrity proof over the canonical Seal using the resolved
   key and declared `cryptosuite`.
5. **Match subject.** Recompute the Subject digest and compare with `subject.digest`.
6. **Check freshness.** If `expires` is present, confirm the current time is at or before it.
7. **Surface compliance.** Return the `compliance` object to the relying party without
   interpreting it as certification.

A Seal is **valid** only if steps 3 through 6 all succeed.

---

## 7. Serialization and media types

| Form | Media type (proposed) |
|------|-----------------------|
| JSON Seal | `application/cyber-seal+json` |
| CBOR Seal | `application/cyber-seal+cbor` |

The JSON form is normative for interoperability. The CBOR form is provided for constrained and
high-volume environments and MUST be a faithful, round-trippable encoding of the JSON form.

---

## 8. Security considerations

- Seals provide **integrity and origin**, not confidentiality. Implementations MUST NOT place
  secrets in a Seal; use the zero-knowledge mode (Section 4.5) when configuration is sensitive.
- Verifiers MUST check revocation state relative to the Seal `created` time, not only the
  current time, to preserve the validity of historical Seals after routine key rotation.
- Digest comparison MUST be constant time to avoid timing side channels.
- Resolvers MUST authenticate DID Documents; a compromised Resolver can substitute keys.

---

## 9. Privacy considerations

- The Subject digest reveals nothing about Subject content, but a Seal links an output to an
  Agent and Operator. Producers SHOULD consider whether that linkage is appropriate.
- Compliance metadata is coarse by design and SHOULD NOT carry personal data.
- The zero-knowledge mode is the RECOMMENDED path when disclosure of configuration or exact
  risk posture is undesirable.

---

## 10. Governance and extensibility

CyberProtocol is stewarded on a neutral, multi-stakeholder basis; no single entity controls the
protocol. New compliance keys, cryptosuites, and entity types are added through registered
profiles so that the core remains small and auditable. The specification is the source of truth;
the reference implementation follows it.

---

## 11. References

**Normative**

- RFC 2119, RFC 8174 (requirement keywords)
- RFC 8785 (JSON Canonicalization Scheme)
- W3C Decentralized Identifiers (DID) Core
- W3C Verifiable Credentials Data Model
- W3C Data Integrity, and the `ecdsa-jcs-2019` / `eddsa-jcs-2022` cryptosuites

**Informative**

- EU AI Act (Regulation (EU) 2024/1689)
- NIST AI Risk Management Framework 1.0
- ISO/IEC 42001:2023
- Multiformats: multibase, multihash

---

*This is a draft technical specification released for public review. Comments and proposals are
welcome via the repository issue tracker.*
