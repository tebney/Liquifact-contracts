# Funding Close Snapshot

The `FundingCloseSnapshot` is a critical piece of the LiquiFact Escrow contract's audit trail. It captures the exact state of the escrow at the moment it transitions from `Open` (0) to `Funded` (1).

## Purpose

This snapshot serves as the **immutable source of truth** for off-chain pro-rata calculations. When an invoice is over-funded (which is allowed by the contract), the full `funded_amount` at the threshold-crossing deposit becomes the denominator for investor share calculations, even when it is greater than `funding_target`.

By capturing this state once and making it immutable, the contract ensures that subsequent actions (like SME withdrawals or settlements) do not shift the relative weight of investor contributions.

## Structure

The snapshot is stored under `DataKey::FundingCloseSnapshot` and contains:

- `total_principal`: The sum of all principal contributed at the moment the funding target was met or exceeded. This equals `InvoiceEscrow.funded_amount` at close and can be greater than `funding_target`.
- `funding_target`: The original target for the invoice.
- `closed_at_ledger_timestamp`: The ledger timestamp when the snapshot was captured.
- `closed_at_ledger_sequence`: The ledger sequence number when the snapshot was captured.

## Lifecycle and Immutability

1. **Before close**: `get_funding_close_snapshot()` returns `None` while the escrow is still open and below target.
2. **Creation**: The snapshot is created during `fund` or `fund_with_commitment` only when `status == 0` and the new `funded_amount >= funding_target`.
3. **Over-funding capture**: If the threshold-crossing deposit overshoots the target, `total_principal` records the full over-funded close amount.
4. **Write-Once**: Once the snapshot is written, the contract's logic prevents it from being updated or overwritten. Later funding attempts are rejected because the escrow is no longer open, and later lifecycle writes do not touch `DataKey::FundingCloseSnapshot`.
5. **Persistence**: The snapshot survives all state transitions, including `settle` and `withdraw`.

## Auditing

Integrators can use the `get_funding_close_snapshot` getter to retrieve this metadata. For historical auditing, the `EscrowFunded` event emitted during the snapshot creation contains the `funded_amount` and `status: 1`, allowing off-chain systems to reconcile the snapshot with the event stream.

The `closed_at_ledger_timestamp` and `closed_at_ledger_sequence` fields are captured from the same ledger as the threshold-crossing funding call. Off-chain indexers should use those fields as the canonical close boundary for pro-rata reporting.

## Security Considerations

1. **Time and Sequence Bounds**: The snapshot captures `env.ledger().timestamp()` and `env.ledger().sequence()`. In Soroban, these are provided by the host environment and are reliable for on-chain time-based logic. Off-chain systems should treat these as the canonical boundaries for the "funded" state transition.
2. **Write-Once Denominator**: `DataKey::FundingCloseSnapshot` is only set if it does not already exist. State transitions such as `settle` and `withdraw` do not recompute the denominator, which prevents later writes from changing investor weights.
3. **State-Machine Misuse**: Funding after close is rejected by the `status == 0` funding guard before contribution or snapshot state can be mutated.
4. **Overflow and Amount Guards**: Funding uses positive amount checks and checked arithmetic before writing `funded_amount` or contribution records.
5. **Token Economics and Assumptions**: As detailed in `escrow/src/external_calls.rs`, this contract strictly assumes standard SEP-41 token mechanics. Malicious, rebasing, or fee-on-transfer (FOT) tokens are **explicitly out of scope** and will trigger safe-failure panics at the balance-check boundaries. This ensures that the `total_principal` captured in the snapshot matches standard token accounting assumptions, preserving the integrity of off-chain payout calculations.

## `partial_settle` and Early Snapshot Capture

In addition to the normal `fund`/`fund_with_commitment` path, the snapshot can also be written
via `partial_settle`, which lets the **SME or Admin** promote an under-target open escrow to
`status == 1` (funded) early.

### How it differs from full funding

| Path | Trigger | `total_principal` |
|------|---------|-------------------|
| `fund` / `fund_with_commitment` | `funded_amount >= funding_target` | Full close amount (may exceed target) |
| `partial_settle` | Explicit SME or Admin call while `status == 0` | Current `funded_amount` (may be less than `funding_target`) |

In the `partial_settle` path the snapshot `total_principal` records whatever `funded_amount`
happens to be at the moment of the call, even if it is far below `funding_target`. All
subsequent pro-rata payout calculations via `compute_investor_payout` use **that** value as
the denominator.

### Write-once guarantee

`partial_settle` contains the same `if !env.storage().instance().has(&DataKey::FundingCloseSnapshot)`
guard used in the `fund` path. After the snapshot is written, `partial_settle` transitions the
escrow to `status == 1`, which means any subsequent `partial_settle` call is rejected by the
`status == 0` pre-condition — the snapshot is therefore stable and cannot be overwritten.

### Test coverage

The following tests in `escrow/src/tests/settlement.rs` cover these scenarios:

| Test | What it verifies |
|------|-----------------|
| `test_partial_settle_writes_correct_snapshot` | `total_principal` and `funding_target` are captured correctly |
| `test_partial_settle_snapshot_not_overwritten` | Snapshot is stable after close; second call panics on the status guard |
| `test_partial_settle_investor_payout_via_snapshot` | `compute_investor_payout` produces correct pro-rata payout using the early snapshot |
| `test_partial_settle_sme_happy_path` | SME can trigger early close |
| `test_partial_settle_admin_happy_path` | Admin can trigger early close |
| `test_partial_settle_unauthorized_caller_panics` | Third-party cannot trigger early close |
| `test_partial_settle_blocked_by_legal_hold` | Legal hold prevents early close |
| `test_partial_settle_rejected_if_not_open` | Non-open status blocks early close |
| `test_partial_settle_auth_required` | `require_auth()` is enforced (verified via `mock_auths(&[])`) |
