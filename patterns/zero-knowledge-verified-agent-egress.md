---
title: Zero-Knowledge Verified Agent Egress
status: emerging
authors: ["Provably (@aural-psynapse)"]
based_on: ["Zero-knowledge proofs", "Trusted-endpoint allow-listing"]
category: "Security & Safety"
source: "https://github.com/ProvablyAI/sourcerykit"
tags: [egress, zero-knowledge-proof, source-of-truth, allow-list, mcp, outbound-verification, runtime-protection]
---

## Problem

An agent can be tricked into making an outbound request that looks fine at the network layer but carries a false claim: a payment to the wrong recipient, an API call with tampered parameters, a tool call whose arguments do not match what the user approved. A domain allow-list or egress firewall lets the call through because the destination is allowed; it never checks whether the *content* of the request is truthful. In regulated or high-value flows, teams need each outbound call to prove it matches a trusted source of truth before it leaves, without exposing that source of truth to the agent or the destination.

## Solution

Put a verification layer in front of every outbound request. Before the call leaves, the agent's claims about it get checked against a source of truth, and a zero-knowledge proof confirms the claims hold without revealing the underlying data. The call only goes out if the proof verifies.

The layer wraps the agent's HTTP path and MCP handoffs:

1. **Intercept**: hook the HTTP libraries the agent already uses, plus MCP tool calls, so every outbound request passes through.
2. **Verify**: check the request's claims against the source of truth and produce a zero-knowledge proof that they match. The proof runs against a backend that holds the source of truth, so neither the agent nor the destination sees it.
3. **Enforce**: block anything whose proof does not verify, and block any endpoint not on the trusted allow-list. Log every outbound call for audit.

```pseudo
on_outbound_request(req):
    proof = prove_against_source_of_truth(req.claims)   # zero-knowledge
    if not verify(proof):        return BLOCK
    if req.endpoint not in allow_list:  return BLOCK
    log(req, proof)
    return ALLOW
```

This is complementary to [Egress Lockdown (No-Exfiltration Channel)](egress-lockdown-no-exfiltration-channel.md): that pattern controls *where* an agent may send data at the network layer, while this one verifies that *what* the agent is sending is true before it goes.

## Evidence

- **Evidence Grade:** medium
- **Most Valuable Findings:**
  - Zero-knowledge proofs let a verifier confirm a claim without seeing the secret behind it, so the source of truth stays private from both the agent and the destination.
  - Hooking at the HTTP and MCP layer catches outbound calls regardless of which framework produced them.
- **Unverified / Unclear:** Proof-generation latency and the operational cost of running the verification backend at high call volumes need more production data.

## How to use it

- Wrap any agent framework that makes outbound HTTP or MCP tool calls (LangChain, CrewAI, MCP servers, custom SDKs).
- Define the source of truth for the flows that matter (approved recipients, order details, policy limits) and a trusted-endpoint allow-list.
- Route outbound calls through the verification layer; treat a failed proof or a non-allowlisted endpoint as a hard block.
- Keep the signed logs for audit and incident review.
- Reference implementation: [SourceryKit](https://github.com/ProvablyAI/sourcerykit), a Python SDK (source-available, BSL 1.1) that hooks the HTTP libraries and pairs with a hosted backend that runs the proof and source-of-truth check.

## Trade-offs

- **Pros:** Catches truthful-looking but false requests that a network allow-list misses; the source of truth stays private via the zero-knowledge proof; framework-agnostic through HTTP and MCP hooks; every call is logged.
- **Cons:** Proof generation adds latency per call; verification runs against a backend rather than fully offline; someone has to define and maintain the source of truth for each flow.

## References

- [SourceryKit](https://github.com/ProvablyAI/sourcerykit)
- [Egress Lockdown (No-Exfiltration Channel)](egress-lockdown-no-exfiltration-channel.md)
