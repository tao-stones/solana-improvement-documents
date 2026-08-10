---
simd: 'TBD'
title: Add Transaction Account Data Allocation Limit
authors:
  - Tao Zhu (Anza)
category: Standard
type: Core
status: Idea
created: 2026-08-07
feature: TBD
---

## Summary

Add a transaction v1 configuration value that specifies the maximum executed
account data growth, in bytes, that the transaction is allowed to perform.

SVM must track executed account data growth during transaction execution and fail
the transaction when a resize would exceed the requested limit. Leaders may use
the requested limit when prioritizing and packing transactions, including in
bankless block building.

Legacy and v0 transactions do not receive a new compute-budget instruction for
this value. They use the active protocol default account data allocation limit.

## Motivation

Account data growth can consume validator resources during transaction
execution. Today, the runtime has a cluster-defined per-transaction account data
growth cap, but legacy and v0 transactions do not declare how much executed
account data growth they intend to use.

That has several drawbacks:

- transactions that do not need account data growth are treated as if they could
  use the default maximum;
- SVM cannot fail a transaction early against a user-declared smaller executed
  growth limit;
- leaders cannot reliably prioritize or reserve block allocation capacity based
  on a transaction's declared account data growth budget;
- bankless leaders do not have a transaction-provided allocation bound for block
  packing decisions.

This proposal lets transactions request only the executed account data growth
budget they need by using transaction v1. Over time, the network can reduce the
default account data allocation limit for transaction formats that do not
explicitly request allocation, potentially to zero.

## Dependencies

This proposal has no hard dependency on the proposal to track executed account
data growth against block limits, but the two proposals are complementary.

The requested transaction allocation limit provides a pre-execution upper bound
for executed account data growth. Executed account data growth accounting
provides the consensus value consumed against the block account data allocation
limit.

## New Terminology

**Requested account data allocation limit** is the maximum number of executed
account data growth bytes that a transaction permits itself to use during
execution.

**Executed account data growth** has the same meaning as in the companion block
limit proposal: cumulative positive account data length increases performed
during transaction execution, without refunding later shrinkage.

## Detailed Design

Add a new transaction v1 configuration field:

```rust
accounts_data_allocation_limit: u32
```

The field is encoded in the txv1 configuration section. This proposal does not
add a new compute-budget instruction for legacy or v0 transactions.

Legacy and v0 transactions cannot request a custom account data allocation
limit. They are assigned the active protocol default.

### Sanitization

The requested account data allocation limit:

- is measured in bytes;
- may be zero;
- must be clamped to the cluster-defined maximum per-transaction account data
  allocation limit;
- must be sanitized according to txv1 transaction configuration rules.

If a txv1 transaction does not request a value, the runtime must use the active
default account data allocation limit.

Legacy and v0 transactions always use the active default account data allocation
limit.

At initial activation, the default should remain the existing maximum
per-transaction account data growth limit to avoid breaking existing
transactions. A later activation may reduce the default to a smaller value, and
eventually to zero, once clients and programs have had time to request explicit
allocation limits.

### SVM Enforcement

SVM must track executed account data growth during transaction execution.

Whenever an account data length increases, SVM must add the positive length
increase to the transaction's executed account data growth counter. Account data
shrinkage must not decrement this counter.

If a resize would cause the executed account data growth counter to exceed
`accounts_data_allocation_limit`, SVM must fail the transaction with an account
data allocation limit error. When possible, SVM should fail before performing the
resize that would exceed the limit. The existing
`InstructionError::MaxAccountsDataAllocationsExceeded` error may be used if it
matches the activated semantics; otherwise a new error should be introduced.

The limit applies to all account data growth paths, including:

- top-level instruction account reallocations;
- CPI account reallocations;
- direct-mapped account data growth;
- built-in program account allocation paths;
- any future runtime-supported account resize paths.

The requested limit is a transaction-wide limit. It is shared across all
instructions and CPI calls in the transaction.

If a transaction fails after performing account data growth that was within its
requested limit, the growth already performed remains part of the SVM-reported
executed account data growth consumed by the companion block-limit proposal.

### Cost Model And Leader Behavior

Before execution, the cost model may use the txv1 requested account data
allocation limit as the transaction's account allocation budget for scheduling,
prioritization, and block packing.

For legacy and v0 transactions, the cost model must use the active default
account data allocation limit.

Leaders may deprioritize transactions that request large account data allocation
limits. This makes allocation-heavy transactions compete for block resources
explicitly, instead of receiving the default maximum budget implicitly.

Bankless leaders may use the txv1 requested allocation limit, or the default for
legacy and v0 transactions, as an upper bound when deciding which transactions
can fit under the block account data allocation limit. Replay must still validate
the block using SVM-reported executed account data growth and the active block
limits.

After execution, SVM-reported executed account data growth should be used for
the consensus block growth counter if the corresponding executed-growth
accounting feature is active.

## Alternatives Considered

### Keep Only The Cluster-Defined Per-Transaction Maximum

The existing maximum prevents a single transaction from growing account data
without bound, but it does not let ordinary transactions declare that they need
little or no allocation. This weakens prioritization and bankless packing.

### Use Compute Unit Limit Only

Compute units are a general execution resource. Account data allocation is a
specific byte-oriented resource with its own block limit and validator impact.
Using only compute units does not provide a direct allocation bound for leaders
or SVM.

### Use Finalized Account Growth Only

Finalized account data growth is useful for state-size accounting, but it is only
known after execution and only for successful transactions. A requested
allocation limit gives SVM and leaders a pre-execution bound for execution-time
allocation work and can stop transactions during execution when they exceed their
declared budget.

### Add A Compute-Budget Instruction For Legacy And V0 Transactions

This would give legacy and v0 transactions the same explicit control as txv1,
but it extends older transaction formats with another runtime configuration
surface. This proposal intentionally keeps precise account data allocation
control in txv1. Legacy and v0 transactions use the active protocol default.

## Impact

Transactions that do not grow account data can request a zero allocation limit.
Those transactions become cheaper to reason about for leaders and bankless block
builders. To request that precise limit, users must submit the transaction as
txv1.

Transactions that create, allocate, or realloc account data must request a
sufficient account data allocation limit when using txv1. If they request too
small a limit, SVM will fail the transaction before or when the next resize would
exceed the limit.

Transactions that request large allocation limits may receive lower scheduling
priority or be less likely to fit in a block, depending on leader policy.

Legacy and v0 transactions receive no precise per-transaction control over this
limit. Users who need precise control must migrate to txv1.

## Security Considerations

This proposal protects validators by bounding transaction-local executed account
data growth according to a transaction-declared limit.

All account data growth paths must be wired into the executed growth counter. If
any resize path bypasses the counter, transactions may exceed their declared
allocation limit without being stopped.

Shrinking account data must not refund executed growth within the same
transaction. Otherwise, a transaction could repeatedly grow and shrink account
data while appearing to stay below the limit.

The value is consensus-relevant because it affects transaction success or
failure. All validators must enforce identical limits for the same transaction
under the same feature set.

## Backwards Compatibility

This is a consensus behavior change and must be feature gated.

If the initial default remains the existing per-transaction maximum, existing
transactions continue to behave as they do today unless they opt into a smaller
limit by using txv1.

Lowering the default in a later activation is intentionally breaking for
transactions that grow account data without requesting an explicit allocation
limit. Such a change should have a separate activation schedule and ecosystem
communication period.

Adding the txv1 transaction configuration field requires corresponding SDK,
transaction parser, RPC, and tooling support.

No new compute-budget instruction is added for legacy or v0 transactions.

## Drawbacks

Txv1 clients that create or grow accounts must estimate and request an
appropriate allocation limit.

Over-requesting allocation may reduce a transaction's priority or make it harder
to pack into a block, while under-requesting allocation may cause execution
failure.

The proposal adds another transaction resource dimension that clients, leaders,
and bankless block builders must consider.
