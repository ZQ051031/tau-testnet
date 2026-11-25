# Bounty #01 programmable payments design

This proposal asks for maintainer review of a shared Tau payout path for [standing payments #34](https://github.com/IDNI/tau-testnet/issues/34), [receive routing #35](https://github.com/IDNI/tau-testnet/issues/35), and [rule-computed payouts #63](https://github.com/IDNI/tau-testnet/issues/63). The same path would run conditional grants, including the 5,000-unit Alice/Bob example. Tau rules express who receives funds, how much, and when. The node supplies verified facts, invokes rules at defined events, and validates and commits their effects atomically.

## Reuse and scope

Reuse the existing payout ABI, policy guard, period arithmetic, replay and persistence machinery from whichever upstream candidate maintainers select. At the reviewed heads, #160 demonstrates grants expressed as Tau rules and rechecks the current payout owner's `o5` policy. Its arrival queue can defer a split past the crediting transaction. #177 and #179 stage successful arrival payouts with the originating transfer. #179 additionally demonstrates guarded updates that preserve lineage and accounting, and its conditional grants retain a dedicated execution path. #178 shows a bounded due-point retry model. The proposal specifies how these pieces fit together for maintainer review before implementation. [#160](https://github.com/IDNI/tau-testnet/pull/160) · [#177](https://github.com/IDNI/tau-testnet/pull/177) · [#178](https://github.com/IDNI/tau-testnet/pull/178) · [#179](https://github.com/IDNI/tau-testnet/pull/179) · [Maintainer criterion](https://github.com/IDNI/tau-testnet/pull/146#pullrequestreview-5995504067)

The host provides three trigger kinds, a rule book, checked settlement, calendar and period inputs, a bounded state record, and an optional seal. Policy-specific decisions belong to the clause and its signed parameters. A grant combines ordinary credits, a sealed account, rule state and scheduled evaluations using those same primitives. Rule visibility and recipient refusal controls are outside this proposal.

## Rule terms and execution

- Bind signed installation, update, removal and execution to the genesis hash and account sequence. Identify each rule by a stable slot plus its signed version. Sign its clause, eligible triggers, activation and end conditions, recipient slots, caps, fee ceiling, and any permitted exception to the outgoing transfer policy. Fixed slots cover automatic payouts. An explicitly open slot accepts a recipient supplied by an eligible executor, subject to key validation and the rule and policy checks.
- Require account authorization and the active policy-admin guard for installation, update and removal. Preserve settlement identities, current-period accounting, and lifecycle totals across in-place updates. Changes to the accounting clock, recipient slot identity, or period anchor require a new lineage. Reject updates that would make already-settled spend exceed a new cap. A seal blocks ordinary spending, fee debits from the account, and edits or cancellation until the signed terminal condition settles. Screen queued approval requests as well as newly submitted ones.
- Each invocation sees the state reached at its position in canonical block order. Inputs include consensus time and height, trigger kind and identity, executor and verified credit facts when present, current spendable balance, current-period totals, and the rule's last committed state. Sign resource bounds and allowed outputs. A clause may produce bounded integer legs, state writes, a next wake-up or a terminal marker. Validate each output before applying any of them.
- The host validates recipient keys, slot and total caps, available funds, and network binding. For outbound payout legs it also applies the payer's current transfer policy, including cooldown and spend totals. Evaluate the legs in signed slot order against temporary accounting and commit only if all pass. Advance the rule meters and applicable sender counters once per committed amount. An explicitly authorized, protected exception can apply to specified rule outputs, such as those of a sealed grant whose ordinary spending is blocked.
- State effects use a fixed number of typed slots per rule, with initial values committed at install. A slot can be write-once for first funding time or explicitly mutable for remaining entitlement or most recent qualifying settlement. The host enforces slot types, size and update authority. Tau interprets the values. An accepted arrival can record event facts without paying any legs, and a clock run can close an empty rule without moving funds.
- Persist conditions, state, counters, wake-up time and identities in the consensus snapshot. Use a canonical encoding with deterministic order, checked integer arithmetic and no node-local clock in settlement. Admission and block apply share shape validation and settlement predicates. Admission against a tip is advisory because transaction order can change the inputs. Replay and reorg restore state and pending due points from the canonical snapshot.

## Triggers, ordering and fees

| Trigger | Execution and limits | Fee |
| --- | --- | --- |
| Explicit transaction | An eligible actor submits a signed execution. The rule verifies any required role and committed event reference. When explicit execution and a clock can settle the same installment, both use the same occurrence identity. | The executor pays the ordinary transaction fee within the signed transaction and rule limits. |
| Consensus clock | After user transactions, run due rules from a consensus index, once per rule per block. Append newly due work in due-time and stable rule ID order. A cancel or edit accepted earlier in the block takes effect before the sweep. | Installation pays the ordinary fee. Automatic installments and refunds have no additional fee. |
| Arrival | Run an eligible credit's policy within the crediting transaction, before another transaction can spend the credit. All credit legs and rule effects share one commit. | The originating transaction pays its normal fee. Routing adds no fee and uses only the gross credit. |

### Atomic arrival

Allow one synchronous arrival rule per account. Aggregate a transaction's eligible credit to each account, then evaluate that account once in canonical recipient order. Base every split on that account's own gross credit and state before the transaction, so another account's forwarded share cannot change its split. An arrival rule can send only to fixed, valid non-self destinations. The host verifies the sum of its legs is at most the gross credit. The net balance is gross minus the legs. A valid zero result leaves the full credit with the recipient. A malformed, over-limit or otherwise invalid required evaluation causes the originating transaction to fail as a whole. An operator timeout cannot determine a consensus outcome. Proposers and validators use a deterministic evaluation budget and deterministic handling of evaluator exhaustion.

The initial eligible class is ordinary user transfer. Reserve an explicit credit class for a future user claim path, if maintainers opt in to that class. Faucet, rewards, fees, refunds and all rule-generated legs are terminal credits. The output of a routing rule never re-enters the arrival evaluator, even in a later block. Record gross, net, each route leg and the existing fee together. A split is part of credit allocation, so it does not increment the receiver's outgoing spend counters. Protected administration governs who may change or remove the split.

Budget arrival invocations before including the crediting transaction. If the block has insufficient budget, leave the whole transaction pending for a later block. Enforce the same inclusion limit on validation and reject a single transaction whose invocation count cannot fit any block. Clock work has a separate budget so arrivals cannot displace it. Start with the candidate limit of 16 invocations per block in each automatic lane, subject to measuring the selected evaluator's cost before activation. Account and rule-count limits, bounded clause size and deterministic evaluator limits bound work at install, admission and apply.

For a royalty on an ordinary sale, Tau reads the verified gross credit and writes its floor-divided share. An explicit payout that claims a recorded sale must bind to a canonical credit for that rule's account, authenticated by the node. Use the canonical credit ID for deduplication across executions. Cap outstanding entries and give explicit claims a signed expiry so the book can prune them. An executor-supplied amount alone cannot attest to a sale or release escrow.

### Periods and recurring payments

Use the applying block's consensus UTC timestamp. [Issue #62](https://github.com/IDNI/tau-testnet/issues/62) describes attested inputs for policies that need outside facts. A future rule can accept a verified claim on an explicit execution without making the calendar or automatic scheduler depend on it.

A civil day starts at 00:00 UTC, an ISO week on Monday at 00:00 UTC, and a civil month on the first at 00:00 UTC. A signed fixed window can instead select an anchor and length. Alice's weeks are seven-day windows starting on December 1. Monthly due points are derived from the original signed day and UTC time. Clamp the day to the last day of a short month and return to the signed day when it exists again. Use half-open windows and define each occurrence by lineage and its index, independently of the submitter and transaction ID.

A standing rule can settle one installment in its signed window. A short balance or failed policy check leaves it retryable during that window. When the window closes, skip the occurrence and count it missed once. Express a final date, installment count or total cap in the signed terms. A rule cannot alter its signed amount, recipients, cadence or fee limit during execution.

Track per-period usage by rule lineage, meter and period index. The accepted set of legs assigned to that meter determines the increase, and the same settlement cannot update it twice. Independent per-recipient caps may use disjoint signed slot meters or independent rules. For several payments intended to share a cap, fund one rule with a shared draw meter. The host advances meters from actual legs. The clause can use a separate slot group for a return to the funder, keeping refunds outside the draw allowance.

## Conditional grant using the same rule

Alice installs a sealed rule on a dedicated account. Its signed terms name Alice as funding source and refund destination, Bob as draw executor, initial principal of 5,000, the December 1 start, and December 29 deadline. Installation schedules the absolute deadline so an unfunded rule can close. A qualifying arrival can bind the funding timestamp and initialize the principal state in the same transaction that credits the account. The clause accepts one funding event from Alice of exactly the declared amount before the deadline and requires the resulting unlock to precede the deadline. Other credits cannot initialize or increase the entitlement. Keep principal and its remaining entitlement separate from any incidental account credit so unrelated money cannot increase the grant or its refund. Use independent tranches if later funding needs its own relative hold.

The grant clause evaluates refund eligibility first. The effective deadline is the earlier of the absolute expiry and any enabled inactivity deadline. Before the first draw, inactivity starts from the effective unlock, which is the latest of funding time, signed start and funding time plus any relative hold. After a successful draw, the inactivity base becomes that draw's consensus time. A failed draw leaves it unchanged. For a refund eligible clock invocation, the rule sends the remaining principal to Alice, zeroes the entitlement and closes the seal in the same commit. Exhausting principal may close the rule with zero refund. A draw in or after the deadline block produces no legs to Bob's payee, even if the refund waits in the clock queue.

Before the deadline, an explicit draw must be by Bob and satisfy every configured condition. The rule checks the effective unlock, signed weekday and hour windows in UTC when present, the per-draw ceiling, the remaining weekly allowance, and the remaining principal. An accepted draw pays the validated recipient Bob supplied, debits the associated entitlement and updates the draw clock and meter atomically. Refund or close outputs remain within generic settlement caps. The grant's draw limits belong in its clause and apply only to draw legs.

The funding and draw evaluations schedule or update a generic one-shot wake-up to the earlier deadline. The host validates wake-ups as future timestamps within the signed scheduling horizon. A due wake-up stays eligible until the clause commits a terminal result or commits a permitted new wake-up. At each block, append newly due wake-ups in canonical order, then process the queue in order. Failed runs move to the tail after the block's attempts so an overdue rule gets at most one attempt in a block and cannot starve other due rules. A failed evaluation does not release a seal. The grant template must prove that every reachable funded state can settle the refund within the configured output bounds. A liveness test must cover queue pressure and repeat failures around the deadline.

### Alice and Bob acceptance trace

Use integer test units and UTC timestamps. Fund 5,000 before December 1, 2026. Set the draw ceiling to 500 per seven-day window from December 1, with an automatic refund due December 29 at 00:00 UTC.

| Event | Expected settlement | Remaining principal |
| --- | --- | ---: |
| Funding by Alice | Credit and initialize the sealed entitlement | 5,000 |
| Nov 30 at 23:59:59 | Reject a draw | 5,000 |
| Dec 1 at 00:00 | Bob pays a shop 500 | 4,500 |
| Dec 7 at 23:59:59 | Reject an extra 1 in the same grant week | 4,500 |
| Dec 8 at 00:00 | Bob pays the shop 500 | 4,000 |
| Dec 15 at 00:00 | Bob pays the shop 500 | 3,500 |
| Dec 29 at 00:00 | Reject a draw and return 3,500 to Alice in the clock sweep | 0 |
| Reapply, restart, and follow the same canonical chain | Identical state with one refund | 0 |

Check exact balances and conserved supply, with funding and explicit draw fees charged to their submitters outside the 5,000 principal. The refund is automatic and fee-free. Add an inactivity variant with an earlier wake-up that moves on a successful draw, a funding-relative 48-hour hold funded after installation, and a weekday/hour restriction.

## Acceptance and activation

| Case | Required evidence before implementation is accepted |
| --- | --- |
| Standing payment | Native Tau and block apply. Monthly 31st clamping and March recovery, owner offline, manual and clock deduplication, missed-window skip, retry after funding, live `o5` freeze, cancellation race, and correct fees and spend accounting. |
| Receive split | Native Tau and block apply. 101 at 20% allocates 20 and 81 with one fee. Later same-block spend sees 81. No second hop. Failed routing commits neither credit nor rule effects. Exhausted arrival budget defers the transaction. Protected updates, snapshot restore and network binding preserve the result. |
| Rule-computed payout | Native Tau and block apply. A verified sale of 101 pays five once. Escrow exercises multiple slots and all-or-none settlement after role, fee, cap and live policy checks. Concurrent updates preserve lineage and period usage. |
| Conditional money | Native Tau and signed test-network proposer/follower run for the Alice/Bob trace. Exercise funding-relative hold, scheduled hours, inactivity reset, same-block refund precedence, and no double payment on replay. Differential tests compare a small independent accounting model across interleaved histories, reorgs, restarts and loaded queues. |
| Activation | Consensus compatibility review, bounded-work measurements, canonical snapshot and activation tests, and full mock and native CI against the selected base. |

The [prior audit](https://github.com/ZQ051031/tau-bounty-2026/blob/f612d9ec254fd9070a5b5880eb96bfa831d6c967/AUDIT.md) records candidate test evidence. The table above defines the remaining proofs for the unified implementation.

## Requested maintainer decision

Please confirm whether the shared payout runner should enforce synchronous credit allocation for #35, use a protected payout exception for sealed grants, and expose bounded rule state plus rule-selected one-shot wake-ups for funding-relative holds and idle refunds. Please identify the upstream payout base to extend. After approval, stage the smallest changes against that base, starting with the standing/arrival settlement contract, followed by conditional rules and the end-to-end proof in the program's stage order. [Bounty design process](https://tau.net/bounty-program)
