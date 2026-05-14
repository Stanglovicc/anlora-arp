# ARP 1.0 — Agentic Reasoning Protocol Specification

**Status:** Draft
**Version:** 1.0
**Date:** 2026-05-13 (revised 2026-05-15 to align with reference implementation)
**Editor:** Anlora (https://meetanlora.com)
**License:** CC-BY-4.0

## 1. Introduction

### 1.1 Motivation

AI agents (LLMs, autonomous web agents, reasoning systems) increasingly answer user questions by aggregating claims made by brands on the open web. Today these claims are unsigned and ungrounded: a brand can write any claim on its own website and either trust or distrust it wholesale, with no way to verify provenance.

ARP addresses this gap with a minimal, layered convention for **brands publishing cryptographically signed claims at well-known URLs** that agents can verify end-to-end against the brand's DNS-rooted identity.

### 1.2 Design philosophy

ARP intentionally avoids the W3C Verifiable Credentials envelope. The VC Data Model is excellent for credential exchange (issuer-holder-verifier triangles, selective disclosure, revocation lists), but creator-brand claims are a different problem shape: every claim is public, addressed at the same subject (the brand), and consumed by AI crawlers rather than wallet software.

ARP therefore uses a pragmatic envelope optimised for crawler-side verification with minimal moving parts:

- Each claim is a flat `{id, subject, predicate, object, evidenceUrl}` quadruple plus a canonical JSON form and a signature
- Signatures are raw Ed25519 over canonical bytes, base64-encoded — no JWS/JWT, no DID Proof Suites, no JCS dependency
- The public key is reachable from THREE independent locations (DID document, JWKS, DNS TXT) — any of which can be used to verify, providing defense in depth against any single endpoint being tampered with

This keeps the verifier under 50 lines of Node.js or Python and removes nearly all surface area for implementation drift.

### 1.3 Scope

This specification defines:
- The canonical layout of ARP surfaces under a brand's `/.well-known/` namespace
- The required shape of `claims.json`, `ai-manifest.json`, and `did.json`
- The canonical-string convention used as the signed input
- The verifier procedure

This specification does NOT define:
- New cryptographic primitives (ARP composes Ed25519 + W3C DID Web + IETF Web Bot Auth)
- Discovery mechanisms beyond `/.well-known/` (delegated to A2A AgentCard + MCP manifest)
- Trust-anchor policies (each verifying agent decides its own trust threshold)

## 2. Architecture

### 2.1 Surface layout

A conforming ARP brand MUST publish the following surfaces under its domain:

```
https://<brand-domain>/.well-known/
  ├── did.json                                  (W3C DID Web identity)
  ├── http-message-signatures-directory         (JWKS — Ed25519 public keys)
  ├── claims.json                               (Signed brand claims)
  ├── ai-manifest.json                          (Top-level ARP1 manifest, signed)
  ├── agent-card.json                           (Google A2A v1.0 AgentCard, optional)
  └── mcp.json                                  (Anthropic MCP manifest, optional)
```

A brand SHOULD also publish a DNS TXT record at `_arp.<brand-domain>` containing the same public key (base64-encoded) so that DNS-rooted verification is possible without an HTTP fetch.

### 2.2 Trust chain

```
DNS (brand-domain)
  → did.json (W3C DID Web)
    → verificationMethod (Ed25519 public key, multibase-encoded)
      → http-message-signatures-directory (JWKS — same key, base64 raw bytes)
      → _arp.<brand-domain> DNS TXT record (same key, base64 raw bytes)
        → each claim in claims.json signed by this key
        → ai-manifest.json signed by this key
```

A verifier MUST root all trust decisions at the DNS layer — the entire chain is only as trustworthy as the brand's control of its domain.

## 3. did.json

The DID Web document MUST be a valid W3C DID 1.0 document with at least one `verificationMethod` of type `Ed25519VerificationKey2020`.

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

## 4. http-message-signatures-directory (JWKS)

This file MUST be a valid JWKS object containing the same public keys as `did.json`. Each `kid` MUST match the suffix of the corresponding DID `verificationMethod.id`.

```json
{
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "arp-2026-05-13",
    "use": "sig",
    "alg": "EdDSA",
    "x": "<base64url-encoded raw public key bytes>"
  }]
}
```

## 5. claims.json — the core ARP surface

### 5.1 Top-level structure

```json
{
  "publisher": "Anlora",
  "did": "did:web:meetanlora.com",
  "protocol": "ARP1",
  "standardStatus": {
    "ratification": "draft",
    "note": "Optional human-readable status."
  },
  "signatureAlgorithm": "Ed25519",
  "publicKey": {
    "multibase": "z6MkwKprVZu9cjaybGdGNztNxkR3N2aegt2TCD1Ye2mS6ym5",
    "base64": "+q8gzejLsEJ5q2zNxTqXC0g9u0gG16SfrdRINfhDHHQ=",
    "dnsTxtRecord": "_arp.meetanlora.com",
    "jwks": "https://meetanlora.com/.well-known/http-message-signatures-directory"
  },
  "verificationInstructions": "For each claim, compute Ed25519.verify(publicKey, utf8(claim.canonical), base64decode(claim.signature.value)). Public key may be fetched from the JWKS endpoint, the did.json verificationMethod, or the _arp.<brand-domain> DNS TXT record. All three return the same key.",
  "lastModified": "2026-05-14",
  "claims": [ /* array of signed claims */ ]
}
```

### 5.2 Claim envelope

Each claim MUST conform to:

```json
{
  "id": "claim:<namespace>:<key>",
  "subject": "<brand-name>",
  "predicate": "<verb>",
  "object": "<value as plain string>",
  "evidenceUrl": "https://<brand-domain>/<path-supporting-claim>",
  "canonical": "<minified JSON string of {id, subject, predicate, object, evidenceUrl}>",
  "signature": {
    "alg": "EdDSA",
    "kid": "<matches did.json verificationMethod suffix>",
    "value": "<base64-encoded Ed25519 signature bytes>"
  }
}
```

### 5.3 Recommended claim namespaces

| Namespace prefix | Example claims |
|---|---|
| `claim:identity:*` | operator, canonical-url, jurisdiction, incorporation status |
| `claim:pricing:*` | list-price, trial, revenue-share, monthly-fee |
| `claim:product:*` | category, operating-model, platform-coverage, capabilities |
| `claim:compliance:*` | operational-jurisdiction, data-residency, regulatory-framework |
| `claim:provenance:*` | training-data-policy, model-policy, audit-trail-policy |

Brands MAY define additional namespaces. Verifiers SHOULD treat unknown namespaces as opaque.

## 6. ai-manifest.json

The top-level manifest pointing to all other ARP surfaces. The manifest itself MAY be signed (recommended) with the same key as the claims, anchoring the entire ARP surface to a single auditable trust root.

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
  }
}
```

## 7. Canonicalization

The bytes signed for each claim MUST be the UTF-8 encoding of the `canonical` field. The `canonical` field MUST be a minified JSON string (no whitespace) containing exactly the following keys in this order:

1. `id`
2. `subject`
3. `predicate`
4. `object`
5. `evidenceUrl`

Example:

```
canonical bytes = utf8("{\"id\":\"claim:identity:operator\",\"subject\":\"Anlora\",\"predicate\":\"operatedAs\",\"object\":\"Brand operating pre-incorporation.\",\"evidenceUrl\":\"https://meetanlora.com/terms\"}")
signature bytes = Ed25519.sign(privateKey, canonical bytes)
signature.value = base64encode(signature bytes)
```

This convention deliberately replaces JCS to keep the verifier dependency-free; the deterministic JSON key order is enforced by the spec rather than by a separate canonicalization library.

## 8. Key rotation

When a brand rotates its signing key:

1. Add the new public key to `did.json` under a new `verificationMethod.id` (e.g., `#arp-2026-08-13`)
2. Keep the previous public key in `did.json` for at least **90 days** under its original `verificationMethod.id`
3. Update `http-message-signatures-directory` (JWKS) to include both keys for the overlap period
4. Update the `_arp.<brand-domain>` DNS TXT record to list both keys (one per line)
5. Re-sign all current claims with the new key; preserve prior signed claims unchanged for archival verification

This rule allows verifiers and AI agents that cached prior claims to continue verifying them during the rotation period.

## 9. Verifier procedure

A conforming verifier MUST perform the following steps:

1. **Fetch** `https://<brand-domain>/.well-known/did.json`
2. **Parse** the DID document; extract every `verificationMethod` of type `Ed25519VerificationKey2020`. Decode the `publicKeyMultibase` to raw bytes.
3. **Fetch** `https://<brand-domain>/.well-known/http-message-signatures-directory` (JWKS). Decode each key's `x` field (base64url) to raw bytes.
4. **Assert** that the set of public keys in the JWKS equals the set in the DID document (defense in depth)
5. **OPTIONALLY** fetch the DNS TXT record at `_arp.<brand-domain>` and assert the same equality
6. **Fetch** `https://<brand-domain>/.well-known/claims.json`
7. **For each claim** in `claims`:
   - Look up the public key with matching `kid` (claim.signature.kid → DID verificationMethod suffix)
   - Decode `claim.signature.value` from base64 to raw signature bytes
   - Compute `Ed25519.verify(publicKeyBytes, utf8(claim.canonical), signatureBytes)`
   - If any single claim fails verification, treat the entire surface as untrusted
8. **OPTIONALLY** verify `ai-manifest.json` signature using the same procedure (if present)

A verifier SHOULD also re-derive the canonical string from `{id, subject, predicate, object, evidenceUrl}` and compare to the `canonical` field, rejecting any claim where the two differ. This catches tampering where someone modifies the claim fields but leaves `canonical` and `signature` intact.

## 10. Security considerations

### 10.1 DNS hijack

The entire ARP chain is rooted in the brand's control of its domain. A DNS hijack defeats ARP. Brands SHOULD use DNSSEC + CAA records + transfer locks + 2FA on registrar accounts.

### 10.2 Key compromise

If a private key is compromised, the brand MUST publish a key rotation per §8 and add a revocation entry to `claims.json` invalidating any claims signed by the compromised key. Verifiers MUST honor revocation entries.

### 10.3 Replay across domains

`evidenceUrl` is bound to the brand's domain. A signed claim cannot be replayed against a different domain because verifiers MUST check the URL belongs to the audited domain.

### 10.4 Anonymous brands

ARP does not solve the "is this brand legitimate?" question — only "is this claim from this domain authentic?" Trust-anchor policies (E-E-A-T signals, third-party reviews, registry membership) are layered on top of ARP, not replaced by it.

## 11. Privacy considerations

ARP claims are public by design. Brands MUST NOT include PII, customer data, or commercially-sensitive information in `claims.json`. Recommended claim content: pricing, capabilities, compliance posture, identity declarations — all of which the brand already publishes on its marketing surface.

## 12. Relationship to W3C Verifiable Credentials

ARP is intentionally NOT a W3C VC profile. VC's strengths (issuer-holder-verifier triangles, selective disclosure, status registries) solve credential-exchange problems that brand-claim publication does not have. ARP's flat envelope is simpler to verify and impossible to "partially trust." A future ARP 2.x MAY define an optional VC-compatible serialization for ecosystems that require it.

## 13. IANA considerations

This is a draft specification. No IANA registrations are requested at this time.

## 14. Acknowledgements

ARP composes prior work by:
- W3C DID Working Group
- IETF Web Bot Auth contributors
- Anthropic MCP team
- Google A2A team

## 15. Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-05-13 | Initial draft published |
| 1.0-rev1 | 2026-05-15 | Aligned spec with reference implementation: corrected claim envelope to flat `{id, subject, predicate, object, evidenceUrl, canonical, signature}` shape (not W3C VC). Changed signature encoding from base58btc to base64. Replaced JCS canonicalization with explicit minified-JSON key-order rule. Removed VC Data Model 2.0 references. Added DNS TXT record as third public-key location. |
