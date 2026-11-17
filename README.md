# inventory-ledger-drift-task

A long-horizon backend task built on Medusa v2's inventory module: keep stock and reservation counters consistent with the reservations behind them under concurrency, crashes and retries, and detect and repair drift when it happens.

- [task/prompt.md](task/prompt.md) is the prompt given to the model.
- [task/verifiers.md](task/verifiers.md) describes how the work is checked.

Before using it, pin the exact Medusa v2 tag and confirm upstream has not already shipped an equivalent fix.
