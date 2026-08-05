# CyberProtocol AI Trust Standard — Specification

**Version 1.0 (Draft, Initial Proposal) · August 2026**

This directory holds the source of truth for the CyberProtocol Standard. The
narrative overview is published as the [Whitepaper](https://cyberprotocol.io/whitepaper.html);
this document records the technical specification.

> **Status:** Draft. Data formats below are proposed and may change before v1.0 is finalized.
> Contributions and review are welcome (see [CONTRIBUTING](../CONTRIBUTING.md)).

---

## 1. Terminology

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be interpreted as
described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

- **Agent** — an AI system that produces outputs under an operator's control.
- **Operator** — the legal entity accountable for an Agent.
- **Seal** — a cryptographic attestation bound to a specific output.
- **Verifier** — any party that validates a Seal without proprietary software.

---

## 2. Pillars

CyberProtocol defines four verifiable layers. A conforming implementation SHOULD
support all four; it MUST clearly declare which pillars it implements.

### Pillar 1 — AI & Human Identity

- Human identities are expressed as W3C Verifiable Credentials.
- AI agents are identified by a Decentralized Identifier using the `cyber` method:
  `did:cyber:agent:<multibase-key>`.
- An agent's DID Document MUST bind a public verification key to the agent's model,
  version, and controlling operator. See [`examples/did-agent.json`](../examples/did-agent.json).

### Pillar 2 — Provenance & Output Certification

- An output MAY carry a `CyberSeal` (see Section 3).
- A Seal MUST reference the digest of the output it certifies.
- A Seal MUST be signed by a key listed in the issuer's DID Document.
- Implementations MAY offer a zero-knowledge mode that proves issuance and compliance
  without revealing configuration or other trade secrets.

### Pillar 3 — Safety & Risk Compliance

- A Seal SHOULD include a `compliance` object mapping to recognized frameworks.
- Defined vocabularies: `euAiAct`, `nistRmf`, `iso42001` (see Section 4).

### Pillar 4 — Cross-Border Verification

- The Seal format MUST NOT depend on any single nation's signature scheme.
- Any party MUST be able to verify a Seal using open tools and public key material.

---

## 3. Data model: the CyberSeal

A Seal is a JSON (or CBOR) object. Required members:

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | MUST be `"CyberSeal"`. |
| `issuer` | DID | The agent that produced the output. |
| `created` | datetime | ISO 8601 timestamp of issuance. |
| `subject.digest` | string | Multihash of the certified output. |
| `proof` | object | A W3C Data Integrity proof over the Seal. |

Optional members: `config.hash` (a commitment to the agent configuration) and
`compliance` (Section 4). A canonical example is in
[`examples/cyber-seal.json`](../examples/cyber-seal.json).

---

## 4. Compliance vocabulary

The `compliance` object maps a single Seal to multiple regulatory frameworks:

| Key | Values (proposed) | Maps to |
|-----|-------------------|---------|
| `euAiAct` | `prohibited`, `high-risk`, `limited-risk`, `minimal-risk` | EU AI Act risk categories |
| `nistRmf` | `governed`, `mapped`, `measured`, `managed` | NIST AI RMF functions |
| `iso42001` | `attested`, `not-attested` | ISO/IEC 42001 attestation |

---

## 5. Verification algorithm (informative)

1. Parse the Seal and resolve `issuer` to its DID Document.
2. Confirm the signing key in `proof` is listed in the DID Document.
3. Verify the Data Integrity proof over the Seal.
4. Recompute the digest of the output and compare with `subject.digest`.
5. Surface the `compliance` object to the relying party.

A Seal is **valid** only if steps 2 through 4 succeed.

---

## 6. Security & privacy considerations

- Key rotation and revocation MUST be expressible in the DID Document.
- Seals are integrity proofs, not confidentiality mechanisms; do not place secrets in a Seal.
- The zero-knowledge mode is the recommended path when configuration is sensitive.

---

## 7. References

- W3C Decentralized Identifiers (DID) Core
- W3C Verifiable Credentials Data Model
- W3C Data Integrity
- EU AI Act · NIST AI RMF · ISO/IEC 42001
