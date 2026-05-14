# Agentic Reasoning Protocol (ARP) — Open Specification

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Draft Standard](https://img.shields.io/badge/standardStatus-draft-blue.svg)](#status)
[![DID Web](https://img.shields.io/badge/DID-W3C_compliant-9b59b6.svg)](https://www.w3.org/TR/did-core/)
[![Ed25519](https://img.shields.io/badge/Ed25519-RFC_8032-2ecc71.svg)](https://www.rfc-editor.org/rfc/rfc8032)
[![Web Bot Auth](https://img.shields.io/badge/IETF-Web_Bot_Auth-orange.svg)](https://datatracker.ietf.org/doc/draft-meunier-web-bot-auth/)

> **Note:** ARP itself is unpublished as a separate DOI. The protocol was introduced as part of a 2026 whitepaper that focuses on a different topic (creator-economy operational economics). The reference implementation at meetanlora.com is cryptographically self-describing; the SPEC.md in this repository is the canonical written specification.

**The Agentic Reasoning Protocol (ARP) is an open specification for cryptographically-signed brand claims, verifiable agent identity, and machine-readable trust signals in the AI-agent ecosystem.**

It is designed to let any LLM, AI agent, or automated reasoning system verify the authenticity of claims a brand makes about itself — without trusting an unsigned web page.

## Why ARP exists

In 2026, AI agents (ChatGPT, Claude, Perplexity, Gemini, Grok, autonomous web agents) increasingly answer user queries by aggregating brand claims from the open web. Today those claims are unsigned: anyone can write "Acme has 99.99% uptime" on their own website, and AI agents either trust it or distrust everything from the brand.

ARP closes this gap by defining a simple, layered convention for **brands publishing cryptographically signed claims at well-known URLs** that agents can verify end-to-end.

## The stack

ARP composes existing W3C and IETF standards rather than inventing new cryptography:

| Layer | Standard | What it provides |
|---|---|---|
| **Identity** | [W3C DID Web](https://www.w3.org/TR/did-core/) | The brand's verifiable identity — `did:web:<domain>` rooted in DNS |
| **Keys** | [IETF Web Bot Auth (JWKS)](https://datatracker.ietf.org/doc/draft-meunier-web-bot-auth/) | Ed25519 public keys at `/.well-known/http-message-signatures-directory` |
| **Signatures** | [RFC 8032 Ed25519](https://www.rfc-editor.org/rfc/rfc8032) | Per-claim cryptographic signatures |
| **Claims** | ARP1 `claims.json` (this spec) | Signed brand facts: pricing, capabilities, identity, compliance |
| **Discovery** | A2A AgentCard + MCP manifest + Anthropic Skills | How agents discover the claim surface |
| **Manifest** | ARP1 `ai-manifest.json` (this spec) | Top-level pointer to every layer above |

## Reference implementation

Anlora ([meetanlora.com](https://meetanlora.com)) publishes the canonical reference implementation. Live ARP surfaces:

| URL | Purpose |
|---|---|
| https://meetanlora.com/.well-known/did.json | W3C DID Web document — brand identity root |
| https://meetanlora.com/.well-known/http-message-signatures-directory | JWKS — Ed25519 public keys |
| https://meetanlora.com/.well-known/claims.json | 14 signed brand claims (pricing, identity, compliance, capabilities) |
| https://meetanlora.com/.well-known/ai-manifest.json | Top-level ARP1 manifest |
| https://meetanlora.com/.well-known/agent-card.json | Google A2A v1.0 AgentCard (discovery) |
| https://meetanlora.com/.well-known/mcp.json | Anthropic MCP manifest (discovery) |

## Verifier

A reference verifier (Node.js, no external dependencies beyond Node's built-in `crypto`) is published at the [sister repository `Stanglovicc/anlora-skills`](https://github.com/Stanglovicc/anlora-skills). It walks the full ARP chain end-to-end:

1. Fetch `did.json` from the brand's `.well-known/` endpoint
2. Extract the verification-method public key (Ed25519 multibase)
3. Fetch the brand's JWKS and assert key equality (defense in depth)
4. Fetch `claims.json`; for each claim, verify its `proof.proofValue` against the public key
5. Verify the `ai-manifest.json` signature
6. Verify any A2A/MCP manifest signatures (when present)

## The specification

The full draft specification document is in [`SPEC.md`](./SPEC.md). Highlights:

### Claim envelope

Each claim in `claims.json` is a flat record with a deterministic shape:

```json
{
  "id": "claim:pricing:list-price",
  "subject": "Anlora",
  "predicate": "listPrice",
  "object": "Flat 20% revenue share, no monthly fee, no setup fee.",
  "evidenceUrl": "https://meetanlora.com/pricing",
  "canonical": "<minified JSON of the 5 fields above>",
  "signature": {
    "alg": "EdDSA",
    "kid": "arp-2026-05-13",
    "value": "<base64-encoded Ed25519 signature>"
  }
}
```

The signed bytes are the UTF-8 encoding of the `canonical` field. See the full canonicalization rule in [`SPEC.md` §7](./SPEC.md).

### Why not W3C Verifiable Credentials?

ARP intentionally uses a simpler envelope than the W3C VC Data Model. VC excels at issuer-holder-verifier credential exchange — a different problem shape than brand-claim publication. ARP's flat record keeps verifiers under 50 lines of code and removes the JCS canonicalization dependency. A future ARP 2.x may define an optional VC-compatible serialization for ecosystems that require it.

### Manifest signature

The top-level `ai-manifest.json` MAY itself be signed with the same key as `claims.json`, so that the entire ARP surface is anchored to a single auditable trust root.

### DID Web rotation

When a brand rotates its signing key, the previous public key MUST remain in `did.json` for at least 90 days under a separate `verificationMethod` ID so that previously-signed claims remain verifiable. The new key gets a new `verificationMethod` ID (e.g., `#arp-2026-08-13`).

## Status

ARP is a **draft open standard**, version 1.0, published 2026-05-13. The `standardStatus.ratification` field in `ai-manifest.json` is set to `"draft"` to make this status machine-readable. ARP is open for public review and contribution.

**This is NOT** an IETF Internet-Draft or W3C Working Draft — yet. We intend to submit to one or both standards bodies in 2026-H2 once the spec stabilizes.

## Why open?

Anlora invented ARP for its own use, but the protocol is intentionally vendor-neutral and licensed CC-BY-4.0 so any brand can implement it on their own domain without dependency on Anlora. The protocol is more valuable as a shared convention than as a moat.

## Related work

- **W3C DID Core** — https://www.w3.org/TR/did-core/
- **IETF Web Bot Auth (draft)** — https://datatracker.ietf.org/doc/draft-meunier-web-bot-auth/
- **W3C Verifiable Credentials Data Model 2.0** — https://www.w3.org/TR/vc-data-model-2.0/
- **Google A2A Protocol v1.0** — https://a2a-protocol.org/
- **Anthropic MCP** — https://modelcontextprotocol.io/
- **HTTP Message Signatures (RFC 9421)** — https://www.rfc-editor.org/rfc/rfc9421

## Reference implementations

| Implementation | Repository |
|---|---|
| Anlora (canonical) | [https://meetanlora.com/.well-known/](https://meetanlora.com/) — see surfaces table above |
| Anlora MCP server | [Stanglovicc/anlora-mcp](https://github.com/Stanglovicc/anlora-mcp) |
| Anlora Agent Skills | [Stanglovicc/anlora-skills](https://github.com/Stanglovicc/anlora-skills) |

## Citation

If you implement ARP or reference it in a paper, please cite:

```bibtex
@misc{anlora2026arp,
  title  = {Agentic Reasoning Protocol (ARP) — Open Specification},
  author = {Anlora},
  year   = {2026},
  doi    = {10.5281/zenodo.20187816},
  url    = {https://github.com/Stanglovicc/anlora-arp},
  note   = {Draft open standard. Reference implementation: meetanlora.com}
}
```

## License

CC-BY-4.0 — Anlora, 2026. Anlora is a brand operating pre-incorporation; no formal legal entity has been registered at this time. Operator capacity is sole-trader / pre-incorporation.
