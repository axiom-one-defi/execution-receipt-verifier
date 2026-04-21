# Execution Receipt Schema (v0.2)

This document describes the receipt format accepted by `POST /verify-receipt`.

The schema is public; any system can produce or independently verify receipts against it without running the hosted API.

## Top-level fields

| Field | Type | Required | Notes |
|---|---|:---:|---|
| `version` | string | ✓ | Receipt schema version. Currently `"0.2"`. |
| `receiptId` | string | ✓ | Unique identifier. Recommended convention: `axm-<type>-<random>`. |
| `chainId` | number | ✓ | EIP-155 chain id of the origin execution (not the settlement chain of the payment). |
| `actionType` | string | ✓ | E.g. `"wallet_guard.execute"`. Determines which per-action fields are required. |
| `status` | string | ✓ | `"executed"` \| `"blocked"`. |
| `evidenceSource` | string | conditional | Required when `status: "blocked"`. E.g. `"precheck"`. |
| `decision` | object | ✓ | See below. |
| `inputs` | object | ✓ | Action inputs (wallet, target, value, selector). |
| `outputs` | object | optional | Structured outputs from the action/guard. Must not contain internal identifiers, PII, or system architecture details — limit to non-sensitive decision evidence (e.g. boolean policy flags, numeric thresholds). |
| `txHash` | string \| null | conditional | Required when `status: "executed"`. Hex tx hash. |
| `blockNumber` | number \| null | conditional | Required when `status: "executed"`. |
| `decisionHash` | string | ✓ | Deterministic hash of the canonical decision payload. Format `<prefix>:<hex>` (e.g. `precheck:…` or `0x…`). |
| `completedAt` | string | ✓ | ISO-8601 timestamp. |
| `guardAddress` | string | conditional | Required when `actionType` is `wallet_guard.execute`. 20-byte contract address. |
| `hash` | string | ✓ | SHA-256 of the full receipt. |

### `decision` object

| Field | Type | Required | Notes |
|---|---|:---:|---|
| `result` | string | ✓ | `"allow"` \| `"deny"`. |
| `reasonCode` | string | ✓ | Machine-parseable code (e.g. `DENY_DESTINATION_NOT_ALLOWED`). |
| `reason` | string | ✓ | Human-readable explanation. |

### `inputs` object (for `wallet_guard.execute`)

| Field | Type | Notes |
|---|---|---|
| `wallet` | string | Address that initiated the call. |
| `target` | string | Destination contract or EOA. |
| `value` | string | Wei-denominated value as decimal string (avoids JS number precision). |
| `selector` | string | 4-byte function selector (`0x…`). |

## Action types

### `wallet_guard.execute`

Represents a Wallet Guard precheck or post-execution record.

**Required extras:** `guardAddress`.

**Example — blocked precheck:**

```json
{
  "version": "0.2",
  "receiptId": "axm-precheck-f0eb4...",
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
    "wallet": "0x7099...",
    "target": "0x0000...",
    "value": "100000000000000000",
    "selector": "0x773acdef"
  },
  "outputs": {
    "policyExists": true,
    "destinationAllowed": false,
    "agentPaused": false
  },
  "txHash": null,
  "blockNumber": null,
  "decisionHash": "precheck:76d48f5139e65...",
  "completedAt": "2026-04-20T08:42:33Z",
  "guardAddress": "0x3aa5...",
  "hash": "33b8c106f466..."
}
```

**Example — executed transaction:**

```json
{
  "version": "0.2",
  "receiptId": "axm-receipt-1",
  "chainId": 84532,
  "actionType": "wallet_guard.execute",
  "status": "executed",
  "decision": {
    "result": "allow",
    "reasonCode": "ALLOW_ALL_CHECKS_PASSED",
    "reason": "Action passed policy checks and was executed."
  },
  "inputs": {
    "wallet": "0x7099...",
    "target": "0xc6e7...",
    "value": "100000000000000000",
    "selector": "0x773acdef"
  },
  "outputs": {},
  "txHash": "0xdcc23d96...",
  "blockNumber": 28,
  "decisionHash": "0x5cce7b9256...",
  "completedAt": "2026-04-20T08:42:33Z",
  "guardAddress": "0x3aa5...",
  "hash": "fed0c6b8712b..."
}
```

## Canonicalization

The `decisionHash` is computed over a **canonical, deterministic** serialization of the receipt's decision-bound fields — not the raw JSON. Key rules:

- object keys sorted lexicographically
- addresses normalized to lowercase hex
- numeric values represented as decimal strings (no scientific notation)
- `null` fields retained in their canonical position
- timestamps kept as ISO-8601 strings

Two receipts that represent the same decision must produce the same `decisionHash`. Any mutation — reordering, case change on `wallet`, different numeric form — changes the hash and trips `decisionHashMatchesReceiptPayload: false`.

## Tampering

The verifier recomputes the `decisionHash` from the received payload and compares. Any mismatch surfaces as:

```json
{
  "valid": false,
  "checks": { "decisionHashMatchesReceiptPayload": false, ... },
  "reasons": ["decisionHash mismatch: expected <...> got <...>"]
}
```

This is why the schema emphasizes canonicalization — the hash is only meaningful if producers agree on the canonical form.

## Extending the schema

New action types are added by specifying their required extras and evidence shape. Until a contributor process is published, open an issue with the intended `actionType` and the evidence your system produces.
