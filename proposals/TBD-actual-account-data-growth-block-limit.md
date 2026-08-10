---
simd: 'TBD'
title: Track Executed Account Data Growth Against Block Limits
authors:
  - Tao Zhu (Anza)
category: Standard
type: Core
status: Idea
created: 2026-08-07
feature: TBD
---

## Summary

Use executed account data growth, as reported by SVM, when tracking the
per-block account data allocation limit.

This replaces static transaction-message inference for the block-level account
data growth dimension with the account data growth that actually took place
during transaction execution. This includes account data growth from transactions
that enter SVM execution but ultimately fail.

## Motivation

Validators currently enforce a per-block limit on account data allocation/growth
through the cost tracker. The cost model estimates this value before execution by
inspecting transactions, especially system-program account allocation
instructions.

That estimate is best-effort and incomplete:

- it cannot accurately account for account data growth performed by arbitrary
  programs;
- it cannot fully account for account data growth performed through CPI;

For example, a transaction may contain two instructions. The first instruction
allocates and zero-fills 10 MiB of account data. The second instruction fails.
The transaction does not commit the 10 MiB account growth, but every validator
that executes the transaction still spent resources performing the allocation
work before the failure.

The goal of this proposal is to make the block account data growth limit track
the number of account data bytes allocated or grown during block execution,
regardless of whether the transaction ultimately succeeds. This better protects
validators from blocks that cause excessive account allocation work and hardens
block cost accounting against underestimation.

## Dependencies

This proposal depends on SVM exposing executed account data growth for executed
transactions, whether the transaction ultimately succeeds or fails.

The SVM execution result must include a value equivalent to:

```rust
executed_account_data_size_growth: u64
```

where the value represents account data growth performed during transaction
execution.

This value must be available for every transaction that enters instruction
execution. It must not be nested under a successful-only account delta structure.

This proposal is complementary to a separate proposal for transaction-requested
account data allocation limits. The requested allocation limit replaces
statically estimated account allocation size for leader scheduling and allows SVM
to stop a transaction once it exceeds its declared allocation budget. This
proposal aggregates the actual executed growth against the block limit.

## New Terminology

**Executed account data growth** is the cumulative positive account data length
increase performed during transaction execution, without refunding later
shrinkage, for every processed transaction whose execution enters SVM. It is
counted whether the transaction ultimately succeeds or fails.

For a single resize operation:

```text
executed_growth += max(new_data_len - old_data_len, 0)
```

If an account grows, then later shrinks, the shrink does not reduce executed
growth. If the account grows again, the later growth is counted again.

## Detailed Design

SVM must calculate executed account data growth for every transaction whose
instruction execution enters SVM.

For each such transaction, SVM must return executed account data growth in bytes.
The value must:

- be zero for transactions that do not grow account data during execution;
- include account data growth from top-level instructions and CPI;
- include account data growth from all runtime-supported account resize paths;
- include account data growth performed before a later instruction failure;
- count only positive growth for each account data resize operation;
- not allow account shrinkage to offset prior or later growth in the same
  transaction;
- fit in a `u64`, using saturating arithmetic where needed.

Transactions that fail before instruction execution begins, such as transactions
that fail to load, fee-only transactions, and no-op transactions, do not perform
account data growth during SVM execution and therefore report zero executed
account data growth.

After feature activation, replay and leader-side cost tracking must use the SVM
reported executed account data growth when updating the block account data
allocation counter.

Specifically:

- `TransactionCost.allocated_accounts_data_size` must be populated from
  SVM-reported executed account data growth for processed transactions.
- `CostTracker` must continue to enforce `allocated_data_size` against the sum
  of `allocated_accounts_data_size` for transactions accepted into the block.
- pre-execution estimates may continue to be used for scheduling or queueing,
  but they must not be the consensus value for the block-level account data
  growth limit after this feature is active.

If adding a transaction's executed account data growth would exceed the block
account data allocation limit, the transaction must be rejected from the block by
leaders. During replay, a block whose cumulative executed account data growth
exceeds the active limit must fail block verification.

## Alternatives Considered

### Keep Static Cost Model Inference

The existing approach is cheaper to compute before execution, but it cannot
accurately represent account growth from arbitrary programs, CPI, or failed
transactions that performed allocation work before failing. It also requires the
cost model to infer runtime behavior from transaction instructions, which is
structurally incomplete.

### Use Signed Net Account Data Delta

The bank already tracks signed account data size changes for total accounts data
size accounting. A signed net value is not appropriate for a block allocation
limit because shrinking account data should not refund allocation work already
performed by validators.

### Count Finalized Growth From Successful Transactions Only

Counting only finalized growth protects state growth, but it does not protect
validators from resources spent on failed transactions. A transaction that grows
account data and then fails still consumes allocation and zero-fill resources
during execution.

## Impact

Blocks containing transactions that grow account data through non-system-program
paths may consume more of the block account data allocation limit than they do
today.

Blocks containing failed transactions that grow account data before failing may
consume more of the block account data allocation limit than they do today.

The resulting block limit more directly corresponds to account allocation work
performed by validators while executing the block.

## Security Considerations

This proposal strengthens validator protection by making the block account data
growth limit depend on SVM-observed execution results rather than static
instruction inference.

All account data growth paths must be included in the SVM-reported value. Missing
an account resize path would undercount block execution resource usage and
weaken the limit.

The value is consensus-relevant. All validators must calculate the same executed
account data growth for the same transaction under the same feature set,
including transactions that fail after entering SVM execution.

## Backwards Compatibility

This is a consensus behavior change and must be feature gated.

Transactions that previously fit in a block because their actual account data
growth was underestimated may no longer fit after activation.

Failed transactions that grow account data before failing may consume block
account data allocation limit after activation.

No transaction wire format change is required by this proposal if SVM can expose
the required value through execution results.

## Drawbacks

Leaders may spend execution resources on a transaction and only later discover
that its executed account data growth does not fit in the remaining block limit.
This is already the general shape of post-execution actual cost tracking, and it
is addressed more directly by the companion SIMD that introduces a
transaction-account-data-allocation-limit.

The proposal requires plumbing a new SVM execution result field through replay,
leader-side banking, cost-model, conformance, and transaction status/cost paths.
