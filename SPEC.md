# ARP 1.0 — Agentic Reasoning Protocol Specification

**Status:** Draft
**Version:** 1.0
**Date:** 2026-05-13
**Editor:** Anlora (https://meetanlora.com)
**License:** CC-BY-4.0
**Citation DOI:** [10.5281/zenodo.20187816](https://doi.org/10.5281/zenodo.20187816)

## 1. Introduction

### 1.1 Motivation

AI agents (LLMs, autonomous web agents, reasoning systems) increasingly answer user questions by aggregating claims made by brands on the open web. Today these claims are unsigned and ungrounded: a brand can write any claim on its own website and either trust or distrust it wholesale, with no way to verify provenance.

ARP addresses this gap with a minimal, layered convention for **brands publishing cryptographically signed claims at well-known URLs** that agents can verify end-to-end against the brand's DNS-rooted identity.

### 1.2 Scope

This specification defines:
- The canonical layout of ARP surfaces under a brand's `/.well-known/` namespace
- The required envelope for each signed claim
- The signature canonicalization rules
- The key rotation policy
- The verifier procedure

This specification does NOT define:
- New cryptographic primitives (ARP composes existing W3C/IETF standards)
- Discovery mechanisms beyond `/.well-known/` (delegated to A2A AgentCard + MCP manifest)
- Trust-anchor policies (each verifying agent decides its own trust threshold)

## 2. Architecture

### 2.1 Surface layout

A conforming ARP brand MUST publish the following six surfaces under its domain:

```
https://<brand-domain>/.well-known/
  ├── did.json                                  (W3C DID Web identity)
  ├── http-message-signatures-directory         (JWKS — Ed25519 public keys)
  ├── claims.json                               (Signed brand claims)
  ├── ai-manifest.json                          (Top-level ARP1 manifest, signed)
  ├── agent-card.json                           (Google A2A v1.0 AgentCard, optional)
  └── mcp.json                                  (Anthropic MCP manifest, optional)
```

### 2.2 Trust chain

```
DNS (brand-domain)
  → did.json (W3C DID Web)
    → verificationMethod (Ed25519 public key, multibase-encoded)
      → claims.json (each claim signed by this key)
      → ai-manifest.json (signed by this key)
      → agent-card.json signature (when present)
      → mcp.json signature (when present)
```

A verifier MUST root all trust decisions at the DNS layer — i.e., the entire chain is only as trustworthy as the brand's control of its domain.

## 3. did.json

The DID Web document MUST be a valid W3C DID 1.0 document with at least one `verificationMethod` of type `Ed25519VerificationKey2020`. Example minimal document:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ],
  "id": "did:web:meetanlora.com",
  "verificationMethod": [{
    "id": "did:web:meetanlora.com#arp-2026-05-13",
    "type": "Ed25519VerificationKey2020",
    "controller": "did:web:meetanlora.com",
    "publicKeyMultibase": "z6MkwKprVZu9cjaybGdGNztNxkR3N2aegt2TCD1Ye2mS6ym5"
  }],
  "authentication": ["did:web:meetanlora.com#arp-2026-05-13"],
  "assertionMethod": ["did:web:meetanlora.com#arp-2026-05-13"]
}
```

The `verificationMethod.id` SHOULD be human-readable and date-versioned (e.g., `#arp-2026-05-13`) so that rotation events are traceable from the URL alone.

## 4. http-message-signatures-directory

This file MUST be a valid JWKS object containing exactly the same public keys as `did.json`. Defense in depth: if either file is tampered with, the inconsistency is detectable.

```json
{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "arp-2026-05-13",
    "use": "sig",
    "alg": "EdDSA",
    "x": "<base64url-encoded raw public key>"
  }]
}
```

The `kid` MUST match the suffix of the DID `verificationMethod.id`.

## 5. claims.json

### 5.1 Top-level structure

```json
{
  "@context": ["https://www.w3.org/2018/credentials/v1"],
  "version": "1.0",
  "publishedAt": "2026-05-13T17:00:00Z",
  "validUntil": "2027-05-13T17:00:00Z",
  "issuer": "did:web:<brand-domain>",
  "claims": [ /* array of signed claims */ ]
}
```

### 5.2 Claim envelope

Each claim MUST be a Verifiable Credential per the W3C VC Data Model 1.1:

```json
{
  "@context": ["https://www.w3.org/2018/credentials/v1"],
  "type": ["VerifiableCredential", "BrandClaim"],
  "issuer": "did:web:<brand-domain>",
  "issuanceDate": "2026-05-14T12:00:00Z",
  "credentialSubject": {
    "id": "https://<brand-domain>",
    "claim:<namespace>:<key>": "<value>"
  },
  "proof": {
    "type": "Ed25519Signature2020",
    "created": "2026-05-14T12:00:00Z",
    "verificationMethod": "did:web:<brand-domain>#<key-id>",
    "proofPurpose": "assertionMethod",
    "proofValue": "<base58btc-encoded Ed25519 signature>"
  }
}
```

### 5.3 Recommended claim namespaces

| Namespace | Example claims |
|---|---|
| `claim:identity:*` | operator, jurisdiction, incorporation status |
| `claim:pricing:*` | revenue-share, monthly-fee, setup-fee, custom-pricing-threshold |
| `claim:capabilities:*` | autonomous-level, supported-platforms, languages |
| `claim:compliance:*` | operational-jurisdiction, data-residency, regulatory-framework |
| `claim:provenance:*` | training-data-policy, model-policy, audit-trail-policy |

Brands MAY define additional namespaces. Verifiers SHOULD treat unknown namespaces as opaque.

## 6. ai-manifest.json

The top-level manifest pointing to all other ARP surfaces. MUST be itself signed by the same key as the claims, anchoring the entire ARP surface to a single auditable trust root.

```json
{
  "protocolVersion": "ARP/1.0",
  "standardStatus": { "ratification": "draft" },
  "publisher": {
    "name": "<brand-name>",
    "did": "did:web:<brand-domain>",
    "website": "https://<brand-domain>"
  },
  "publishedAt": "2026-05-13T17:00:00Z",
  "validUntil": "2027-05-13T17:00:00Z",
  "surfaces": {
    "did": "https://<brand-domain>/.well-known/did.json",
    "jwks": "https://<brand-domain>/.well-known/http-message-signatures-directory",
    "claims": "https://<brand-domain>/.well-known/claims.json",
    "agentCard": "https://<brand-domain>/.well-known/agent-card.json",
    "mcp": "https://<brand-domain>/.well-known/mcp.json"
  },
  "proof": { /* same envelope as claims */ }
}
```

## 7. Signature canonicalization

The bytes signed MUST be:

1. The JSON document with `proof.proofValue` set to the empty string `""`
2. Canonicalized per [RFC 8785 JSON Canonicalization Scheme (JCS)](https://www.rfc-editor.org/rfc/rfc8785)
3. UTF-8 encoded

This eliminates ambiguity arising from whitespace, key ordering, or numeric formatting that would otherwise break cross-implementation verification.

## 8. Key rotation

When a brand rotates its signing key:

1. Add the new public key to `did.json` under a new `verificationMethod.id` (e.g., `#arp-2026-08-13`)
2. Keep the previous public key in `did.json` for at least **90 days** under its original `verificationMethod.id`
3. Update `http-message-signatures-directory` to include both keys for the overlap period
4. Re-sign all current claims with the new key; preserve prior signed claims unchanged for archival/replay verification

This rule allows verifiers and AI agents that cached prior claims to continue verifying them during the rotation period.

## 9. Verifier procedure

A conforming verifier MUST perform the following steps:

1. **Fetch** `https://<brand-domain>/.well-known/did.json`
2. **Parse** the DID document; extract every `verificationMethod` of type `Ed25519VerificationKey2020`
3. **Fetch** `https://<brand-domain>/.well-known/http-message-signatures-directory`
4. **Assert** that the set of public keys in the JWKS equals the set in the DID document (defense in depth)
5. **Fetch** `https://<brand-domain>/.well-known/claims.json`
6. **For each claim**: extract `proof.verificationMethod`, look up the matching public key, JCS-canonicalize the claim with `proof.proofValue=""`, verify the Ed25519 signature
7. **Fetch** `https://<brand-domain>/.well-known/ai-manifest.json` and verify its signature using the same procedure
8. **OPTIONALLY** fetch `agent-card.json` and `mcp.json` and verify any signatures present

A verifier MUST treat a single signature failure as a verification failure for the entire ARP surface (a partially-trustable brand is more dangerous than a fully-distrustable one).

## 10. Security considerations

### 10.1 DNS hijack

The entire ARP chain is rooted in the brand's control of its domain. A DNS hijack defeats ARP. Brands SHOULD use DNSSEC + CAA records + transfer locks + 2FA on registrar accounts.

### 10.2 Key compromise

If a private key is compromised, the brand MUST publish a key rotation per §8 and add a revocation entry to `claims.json` invalidating any claims signed by the compromised key. Verifiers MUST honor revocation entries.

### 10.3 Replay across domains

`credentialSubject.id` and `issuer` are both bound to the brand's domain. A signed claim cannot be replayed against a different domain because verifiers MUST check both fields match the domain being audited.

### 10.4 Anonymous brands

ARP does not solve the "is this brand legitimate?" question — only "is this claim from this domain authentic?" Trust-anchor policies (E-E-A-T signals, third-party reviews, registry membership) are layered on top of ARP, not replaced by it.

## 11. Privacy considerations

ARP claims are public by design. Brands MUST NOT include PII, customer data, or commercially-sensitive information in `claims.json`. Recommended claim content: pricing, capabilities, compliance posture, identity declarations — all of which the brand already publishes on its marketing surface.

## 12. IANA considerations

This is a draft specification. No IANA registrations are requested at this time.

## 13. Acknowledgements

ARP composes prior work by:
- W3C DID Working Group
- W3C Verifiable Credentials Working Group
- IETF Web Bot Auth contributors
- Anthropic MCP team
- Google A2A team

## 14. Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-05-13 | Initial draft published |
