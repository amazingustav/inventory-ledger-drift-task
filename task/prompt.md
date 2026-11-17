# Task prompt

Repository: [medusajs/medusa](https://github.com/medusajs/medusa) (MIT), pinned to a v2 release tag.

---

We keep getting bitten by inventory counters drifting in Medusa's inventory module. Each inventory level has `stocked_quantity` and `reserved_quantity`, but the reservation items plus the stock changes on a level are the ledger of record, and the counters should always match them. With concurrent checkouts, or when the server dies mid-workflow, they don't, and nobody notices until stock is wrong. I want that silent drift gone, and caught and fixed when it slips through anyway.

Concretely:

- With no workflow running, `reserved_quantity` is the sum of live reservation items, and `stocked_quantity` is the starting quantity plus every admin or fulfillment change.
- 50 carts checking out at once for N units at one location, backorders off: N orders get placed, the rest get the usual insufficient-inventory error and leave no reservation.
- Kill the server anywhere while placing, cancelling or fulfilling an order. After restart the counters match, and there's no reservation left for an order that never got placed.
- A retried workflow step never reserves or releases twice.
- `GET /admin/inventory-items/:id/ledger-audit` returns, per location, `expected_reserved`, `actual_reserved`, `expected_stocked` and `actual_stocked`.
- `POST /admin/inventory-ledger/reconcile` fixes drifted counters and returns what it changed, with old and new values. Run it again and it returns an empty list.

Anything a single user does today should behave exactly as it does now. Postgres only, and schema changes go in a module migration that runs fine on a database that already has orders.
