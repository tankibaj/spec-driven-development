# Rule: Keep status.yaml Current

After every significant implementation step, update `status.yaml` `last_checkpoint`
with a short description of what was just completed.

This is the recovery point for interrupted sessions. If you forget to update it,
the next session repeats your work from the last recorded checkpoint.
