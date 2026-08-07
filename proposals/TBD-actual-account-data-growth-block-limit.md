---
simd: 'TBD'
title: Track Actual Account Data Growth Against Block Limits
authors:
  - Tao Zhu (Anza)
category: Standard
type: Core
status: Idea
created: 2026-08-07
feature: TBD
---

## Summary

Use actual post-execution account data growth, as reported by SVM, when tracking
the per-block account data allocation limit.

This replaces static transaction-message inference for the block-level account
data growth dimension with the actual finalized growth produced by successfully
executed transactions.

## Motivation

Validators currently enforce a per-block limit on account data allocation/growth
through the cost tracker. The transaction cost model estimates this value before
execution by inspecting transactions, especially system-program account
allocation instructions.

That estimate is incomplete:

- it cannot accurately account for account data growth performed by arbitrary
  programs;
- it cannot fully account for account data growth performed through CPI;
- it does not provide a consensus-checked measurement of finalized account data
  growth per block.

The goal of this proposal is to make the block account data growth limit track
the actual number of account data bytes newly committed in a block. This better
protects validators from blocks that cause excessive finalized account data
growth and hardens block cost accounting against underestimation.

## Dependencies

This proposal depends on SVM exposing actual account data growth for successfully
executed transactions.

The SVM execution result must include a value equivalent to:

```rust
account_data_size_delta: u64
```

where the value represents account data growth finalized by a successful
transaction.

This proposal is complementary to a separate proposal for transaction-requested
account data allocation limits. The requested allocation limit protects SVM
during transaction execution; this proposal protects the finalized block state
growth limit after execution.

## New Terminology

**Finalized account data growth** is the sum, over all accounts committed by a
successful transaction, of positive increases in effective account data length:

```text
sum(max(post_effective_data_len - pre_effective_data_len, 0))
```

Shrinking one account does not offset growth in another account for this block
limit.

An account's effective data length is the amount of account data that contributes
to the bank's on-chain accounts data size after transaction execution. Accounts
that become uninitialized by the end of transaction execution have effective
post-execution data length zero for this calculation.

## Detailed Design

SVM must calculate finalized account data growth for every transaction whose
execution status is successful.

For each successful transaction, SVM must return the finalized account data
growth in bytes. The value must:

- be zero for transactions that do not increase any committed account data size;
- include account data growth from top-level instructions and CPI;
- include account data growth from all runtime-supported account resize paths;
- count only positive finalized account data growth per account;
- not allow account shrinkage to offset account growth elsewhere in the same
  transaction;
- fit in a `u64`, using saturating arithmetic where needed.

For transactions whose execution status is not successful, SVM may return no
account data growth value, or it may return zero. Failed transactions do not
commit account data growth and therefore do not consume this finalized-growth
block limit.

After feature activation, replay and leader-side cost tracking must use the SVM
reported finalized account data growth when updating the block account data
allocation counter.

Specifically:

- `TransactionCost.allocated_accounts_data_size` must be populated from actual
  SVM execution output for processed transactions.
- `CostTracker` must continue to enforce `allocated_data_size` against the sum
  of `allocated_accounts_data_size` for transactions accepted into the block.
- pre-execution estimates may continue to be used for scheduling or queueing,
  but they must not be the consensus value for the block-level finalized account
  data growth limit after this feature is active.

If adding a successfully executed transaction's finalized account data growth
would exceed the block account data allocation limit, the transaction must be
rejected from the block by leaders. During replay, a block that contains
transactions whose cumulative finalized account data growth exceeds the active
limit must fail block verification.

## Alternatives Considered

### Keep Static Cost Model Inference

The existing approach is cheaper to compute before execution, but it cannot
accurately represent account growth from arbitrary programs or CPI. It also
requires the cost model to infer runtime behavior from transaction instructions,
which is structurally incomplete.

### Use Signed Net Account Data Delta

The bank already tracks signed account data size changes for total accounts data
size accounting. A signed net value is not appropriate for a block allocation
limit because shrinking one account should not allow unrelated account growth to
avoid the block growth cap.

### Count Attempted Growth From Failed Transactions

Counting attempted growth from failed transactions would better capture transient
execution-time resource use, but it requires a different SVM reporting contract.
This proposal intentionally targets finalized block state growth. A separate
transaction-requested allocation limit is the better mechanism for bounding
transient allocation work during execution.

## Impact

Blocks containing transactions that grow account data through non-system-program
paths may consume more of the block account data allocation limit than they do
today.

Blocks containing transactions whose statically inferred allocation is larger
than their actual finalized account data growth may consume less of the block
account data allocation limit than they do today.

The resulting block limit more directly corresponds to finalized account data
growth committed by the block.

## Security Considerations

This proposal strengthens validator protection by making the block account data
growth limit depend on SVM-observed execution results rather than static
instruction inference.

All account data growth paths must be included in the SVM-reported value. Missing
an account resize path would undercount block growth and weaken the limit.

The value is consensus-relevant. All validators must calculate the same finalized
account data growth for the same transaction under the same feature set.

## Backwards Compatibility

This is a consensus behavior change and must be feature gated.

Transactions that previously fit in a block because their actual account data
growth was underestimated may no longer fit after activation.

No transaction wire format change is required by this proposal if SVM can expose
the required value through execution results.

## Drawbacks

This proposal does not charge failed transactions for account data growth that
was attempted and then rolled back. Such transactions do not increase finalized
block state, but they may still consume execution-time resources.

Leaders may spend execution resources on a transaction and only later discover
that its actual finalized account data growth does not fit in the remaining
block limit. This is already the general shape of post-execution actual cost
tracking.

The proposal requires plumbing a new SVM execution result field through replay,
leader-side banking, cost-model, conformance, and transaction status/cost paths.
