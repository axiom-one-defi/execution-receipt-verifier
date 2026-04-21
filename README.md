# Execution Receipt Verifier

Public profile for Axiom One's **Execution Receipt Verifier API** — a hosted, paid-per-call verification endpoint for agent execution receipts.

**Live:** [`https://verifier.axiom-one.tech`](https://verifier.axiom-one.tech/health) · Base Sepolia today · [Website](https://axiom-one.tech/verifier)

---

## What it does

Agents take real actions — transactions, blocked prechecks, policy decisions. Teams integrating those agents need **verifiable proof** that what the agent says happened actually happened, without trusting the agent's own log.

This service accepts an execution-receipt JSON and returns a structured verdict:

- schema + canonical-field validation
- deterministic `decisionHash` recomputation and match
- on-chain transaction confirmation (for executed receipts)
- blocked/precheck evidence shape validation

All checks are deterministic and independently reproducible. The hosted API is convenience — the verification logic is the product.

## Endpoint

```
POST https://verifier.axiom-one.tech/verify-receipt
Content-Type: application/json
```

Health check (unauthenticated):

```
GET https://verifier.axiom-one.tech/health
→ {"ok":true,"service":"execution-receipt-verifier-api",
    "network":"eip155:84532","routes":["POST /verify-receipt"]}
```

## Payment flow (x402 on Base Sepolia)

1. Call `POST /verify-receipt` without payment proof → **HTTP 402** with a `payment-required` header containing x402 requirements.
2. Your agent or any x402-compatible client pays via [x402](https://x402.org) (EIP-3009 USDC transfer on Base Sepolia).
3. Retry the POST with the payment proof header → **HTTP 200** with the structured verdict (or **HTTP 422** if the receipt is tampered/invalid).

Testnet calls settle in Base Sepolia tUSDC — no real spend while we validate mainnet. Mainnet access on request.

## Quickstart

### 1. Observe the 402 challenge

```bash
curl -sv -X POST https://verifier.axiom-one.tech/verify-receipt \
  -H "Content-Type: application/json" \
  -d '{}'
```

You'll see:

```
HTTP/2 402
payment-required: <base64-encoded x402 payment requirements>
```

Decode the `payment-required` header (base64 → JSON) to inspect the accepted schemes, network, and recipient.

### 2. Send a well-formed receipt

See [`schema.md`](schema.md) for the full receipt format. A minimal blocked-precheck example:

```json
{
  "version": "0.2",
  "receiptId": "demo-precheck-01",
  "chainId": 84532,
  "actionType": "wallet_guard.execute",
  "status": "blocked",
  "evidenceSource": "precheck",
  "decision": {
    "result": "deny",
    "reasonCode": "DENY_DESTINATION_NOT_ALLOWED",
    "reason": "Destination is not allowlisted."
  },
  "inputs": {
    "wallet": "0x...",
    "target": "0x...",
    "value": "0",
    "selector": "0x00000000"
  },
  "decisionHash": "precheck:<sha256-of-canonical-payload>",
  "guardAddress": "0x...",
  "hash": "<sha256-of-full-receipt>"
}
```

Required for `wallet_guard.execute`: `guardAddress` in the payload. Contract-bound emitter checks depend on it.

### 3. Pass / fail response

On success (`HTTP 200`):

```json
{
  "valid": true,
  "checks": {
    "schemaValid": true,
    "canonicalFieldsPresent": true,
    "txHashConfirmedOnChain": "not-applicable",
    "decisionHashMatchesReceiptPayload": true,
    "blockedPrecheckEvidenceShapeValid": true,
    "verifierExitCode": 0
  },
  "checksRan": [
    "schemaValid",
    "canonicalFieldsPresent",
    "txHashConfirmedOnChain",
    "decisionHashMatchesReceiptPayload",
    "blockedPrecheckEvidenceShapeValid",
    "verifierExitCode"
  ],
  "reasons": []
}
```

On tamper (`HTTP 422`):

```json
{
  "valid": false,
  "checks": { "decisionHashMatchesReceiptPayload": false },
  "reasons": [
    "decisionHash mismatch: expected precheck:a1b2... got precheck:76d4..."
  ]
}
```

## Trust model

The verification logic is open and inspectable. This repository publishes:

- [`schema.md`](schema.md) — receipt format documentation (fields, types, required-by-action-type)
- inline examples for blocked-precheck and executed-tx receipts
- a tampered example to show what the verifier detects

The hosted API is paid convenience; the schema and checks are public so any team can reproduce a verdict locally if they prefer independence over convenience.

## Status

- **Base Sepolia:** live, paid-per-call at $0.02 USDC per verification
- **Mainnet:** on request after validation
- **Scope today:** verification only (checker side)
- **Under discussion:** a producer/writer surface (emit receipts directly from agents/guards) — bounded to keep the verifier independent

## Contact

- Website: [axiom-one.tech](https://axiom-one.tech)
- Audit work: [axiom-one-defi/audit-portfolio](https://github.com/axiom-one-defi/audit-portfolio)
- Email: info@axiom-one.tech

## License

Docs in this repo: MIT.
