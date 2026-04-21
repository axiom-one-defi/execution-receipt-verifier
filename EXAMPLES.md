# Example Receipts

This repository includes three public-safe example receipts under [`examples/`](examples):

- `valid-blocked-receipt.json`: a blocked precheck receipt fixture that should verify as structurally valid.
- `valid-executed-receipt.json`: an executed receipt fixture that should verify with a matching decision hash.
- `tampered-receipt.json`: a deliberately altered fixture used to show invalid verification outcomes.

## Safety Note

These fixtures use synthetic/local test values (Hardhat/local `chainId` 31337, non-production addresses, and fixture hashes) and contain no private keys or customer data.
