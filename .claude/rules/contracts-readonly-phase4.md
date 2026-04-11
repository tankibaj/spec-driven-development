# Rule: Contracts Are Read-Only During Phase 4

During implementation, treat `contracts/` as read-only.
If the code doesn't match the contract, the code is wrong — not the contract.

If implementation reveals a contract issue:

1. **STOP** implementation immediately.
2. Add a blocker to `status.yaml` with a clear description of the mismatch.
3. Set `phase_4.WP-XXX.status` to `blocked`.
4. **Provide a recommendation** to the human. Include:
   - What the contract currently says vs. what the implementation needs
   - Whether the fix is additive (safe) or breaking (needs ADR)
   - Which consumers would be affected by the change
   - Your recommended resolution (e.g., "add optional field X to the Order response schema — additive, no consumers affected")
5. Wait for the human to decide. Do not modify contracts to match your code.
