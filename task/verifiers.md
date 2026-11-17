# Verifiers

Every check runs against a fresh Medusa server built from the attempt, on its own PostgreSQL. The verifier talks to the Admin and Store APIs over HTTP and reads the database directly with a read-only role when it needs the ground truth. It never imports the attempt's code or asserts on private names. Each check maps to one sentence in the prompt.

The verifier keeps its own expected ledger of record: the creation quantity and every stock change it made through the API. It compares that against the stored counters instead of trusting the attempt's audit endpoint.

## Basics

`check_counters_match_reservations_holds`: create items at two locations, run a mixed script of reservations, adjustments, orders, cancellations and fulfillments one at a time. After each step, `reserved_quantity` equals the SQL sum of live reservation items, and `stocked_quantity` equals the verifier's expected value.

`check_audit_endpoint_shape_holds`: the audit endpoint returns one entry per location with the four named fields, and expected equals actual on a clean database.

## Parity checks

`check_single_user_parity_holds`: replay a recorded script of about 40 single-user Admin and Store API calls against the pinned upstream tag and against the attempt. Compare status codes and response bodies after removing ids and timestamps. Any difference fails.

`check_migration_parity_holds`: load a database dump made by the upstream tag with existing orders and reservations, run the attempt's migrations, then rerun the parity script. The audit shows no drift on the migrated data.

## Concurrency

`check_no_oversell_holds`: for N in {1, 7, 30}, create an item with N units, prepare 50 carts each buying one unit, and complete them all at once. Exactly N orders exist, the other 50 − N requests got the insufficient-inventory error, `reserved_quantity` is N, and no reservation item points at a cart without an order. Run it 5 times; every run must pass.

## Failure paths

`check_crash_recovery_holds`: put a TCP proxy (toxiproxy) between Medusa and PostgreSQL. For each of order placement, cancellation and fulfillment, start the operation, then `SIGKILL` the server after a delay drawn from 0 to 2000 ms, or cut the DB connection mid-transaction. Restart, wait for workflow recovery, and compare the stored counters with the verifier's ledger of record and with the SQL sum of reservations. No reservation may exist for a cart without an order. 30 runs per operation.

`check_step_retry_idempotence_holds`: drop the connection right after the reservation step's transaction commits but before its response reaches Medusa, forcing the engine to retry the step. Reserved units for that order equal the ordered quantity, not double.

## Detection and repair

`check_reconcile_repairs_drift_holds`: with orders in place, corrupt counters on three levels directly in SQL (raise one, lower one, change `stocked_quantity` on a third). The audit endpoint reports exactly those three levels with the right expected values. Reconcile returns those three with correct old and new values, the counters now match the ledger of record, and a second reconcile returns an empty list. Untouched levels never appear in any of these responses.

## Resisting a pass without the work

- An audit endpoint that copies `actual_*` into `expected_*` fails `check_reconcile_repairs_drift_holds`, because the verifier corrupts the counters and expects the endpoint to see it.
- A reconcile that rewrites every counter from the reservation table fails, because it must report only the drifted levels and the `stocked_quantity` corruption needs the stock-change history to undo.
- Rejecting every concurrent checkout, or serialising all checkouts behind a global lock that times out, fails `check_no_oversell_holds`: exactly N must succeed, and a timeout is not the insufficient-inventory error.
- Doing nothing fails the concurrency and repair checks, and changing behaviour to dodge them fails the parity checks.
- The verifier owns the client, the proxy, the database role and the expected ledger, so editing the repository's own tests or scripts changes nothing.
