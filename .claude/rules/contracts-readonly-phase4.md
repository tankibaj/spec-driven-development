# Rule: Contracts Are Read-Only During Phase 4

During implementation, treat `contracts/` as read-only.
If the code doesn't match the contract, the code is wrong — not the contract.

If implementation reveals a contract issue: STOP. Add a blocker to `status.yaml`. Ask the human.
Do not modify contracts to match your code.
