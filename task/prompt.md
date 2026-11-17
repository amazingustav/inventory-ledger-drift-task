# Task prompt

Repository: [medusajs/medusa](https://github.com/medusajs/medusa) (MIT), pinned to a v2 release tag.

---

Medusa's inventory module stores the same fact twice. Reservation items record who holds which units, and each inventory level keeps `stocked_quantity` and `reserved_quantity` counters. Treat the reservation items, together with every stock change made to a level, as the ledger of record. The counters must always be derivable from it. Today they can fall out of step under concurrent checkouts or a crash in the middle of a workflow, and nothing notices. I want that silent drift prevented, and detected and repaired when it happens anyway.

What I need:

- Whenever no workflow is running, every inventory level's `reserved_quantity` equals the sum of its live reservation items. Its `stocked_quantity` equals the quantity it was created with plus every change made through the admin API or fulfillment.
- When 50 carts complete at once for an item with N units at one location (backorders off), exactly N orders are placed. The others fail with the existing insufficient-inventory error and leave no reservation behind.
- If the server process is killed at any point while placing, cancelling or fulfilling an order, the ledger of record and the counters agree after restart, and no reservation exists for an order that was not placed.
- A workflow step that is retried after it already succeeded must not reserve or release units a second time.
- `GET /admin/inventory-items/:id/ledger-audit` returns, per location, `expected_reserved`, `actual_reserved`, `expected_stocked` and `actual_stocked`, computed from the ledger of record and the stored counters.
- `POST /admin/inventory-ledger/reconcile` sets every drifted counter back to the value the ledger of record implies and returns the levels it changed with old and new values. A second call right after returns an empty list.
- Single-user flows (creating items, adjusting stock, reserving, placing, cancelling and fulfilling orders) return the same responses and status codes as they do today.

Keep PostgreSQL as the only datastore. Any schema change has to ship as a module migration that works on a database with existing orders.
