# Tamper-evident audit ledger

The security API records confirmed administrator audit events in a per-contract chain. The affiliate payout route records queued and rejected payout requests in a per-wallet chain. Each entry includes a server timestamp, sequence number, event category, producer/actor, payload, previous hash, and SHA-256 hash over RFC 8785 canonical JSON with the `trellis.audit-ledger.v1` domain prefix.

`POST /api/security` keeps its existing `record-audit` and `audit-history` response fields. Audit-history responses add a `ledger` export containing the entries for that contract, the chain head, and a verification result. The `ledger-export` action can export a selected stream for maintainer review. Consumers should verify the export before relying on it.

## Failure handling and recovery

- A sequence or hash mismatch invalidates the stream from the first bad entry. Preserve the export and investigate; do not silently truncate or rebuild the chain.
- Sequence numbers define order. Server timestamps are forced to increase within a stream when the system clock repeats or moves backwards.
- Payout replays do not append duplicate `queued` events. Rejected payout attempts are recorded with a stable error code and no provider error text.
- The current storage is process-local memory, matching the existing security and affiliate stores. Restarting the process or routing requests to another instance can yield a shorter or different chain. This is useful for tamper detection within an instance lifetime, but is not durable compliance storage.
- Before production use for financial controls, persist entries in an append-only database transaction and store signed or externally anchored head hashes. Keep exports and the last trusted head outside the application process.
- For recovery, retain the original export, compare its head with the last trusted checkpoint, and restore from an independently verified copy. If no trusted checkpoint exists, report the gap and start a new versioned stream; never claim the reconstructed history is complete.
- To roll back this feature, first export and retain each stream. Rolling back application code does not restore process-local entries.

The existing audit record and payout response schemas remain unchanged except for additive ledger metadata in the security API responses. Wallet destinations, signatures, and raw provider errors are not added to ledger payloads.
