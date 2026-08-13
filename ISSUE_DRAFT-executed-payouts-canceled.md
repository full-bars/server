# Issue draft: urnetwork/server (FINAL — merged from independent review)

Title: Hung-payment canceler removes payments whose transfer already landed on-chain, with no signal to the client

## Summary

My provider account had a $72.07 payout that was visible as pending on 2026-08-09 and gone from `/account/payments` by 2026-08-13. The wallet's on-chain history shows the transfer landed at 2026-08-02T06:07:00Z (USDC +72.070077, tx below). The server side never recorded completion for it: `tx_hash` stayed empty and `completed` stayed false. `CancelHungAccountPayments` (`model/account_payment_model.go:698`) cancels any payment where `NOT completed AND NOT canceled AND create_time < now()-30d`, with no check of whether the transfer's `payment_record` was ever set or whether the money moved. `GetNetworkPayments` (`model/account_payment_model.go:787`) filters `canceled = false`, so once the sweep fires the row disappears from the API entirely, with no canceled-status row, no notification, and the account total silently drops by that amount.

> [!NOTE]
> This is new code, not an old regression that only recently got scheduled. Both `CancelHungAccountPayments` and its 24h self-rescheduling task were added together in commit `bb4d0676` (2026-07-12), which is in main. I initially assumed (based on an unrelated May 2025 commit that also touches payment cancellation, `urnetwork/server#231`) that the canceler had existed for over a year as dead code; `git log -S` shows that's wrong; the function did not exist before `bb4d0676`. The sweep has only ever run since mid-July 2026, so the first wave of 30-day-old hung payments is only now hitting it.

## Reproduction

Account wallet: `BXqg85kyR4iMJjJwoPGZWfoPtdmoTTDE22drdmYPiLH8` (Solana, USDC payouts).

- 2026-08-09 dashboard/API capture: 71 payments, including a pending $72.07 / 253.27 GB / 29,537 points / 13,314 reliability row.
- 2026-08-13 official API pull: 70 payments. That row is gone; no canceled row appears in its place.
- On-chain, the same wallet shows the transfer completed:
  https://solscan.io/tx/UAQjPZHhpVUJTjqgSct3AfNZwfoRFB71jmjjz3rfV4LPVywDz5jeoEjVwxXTazxfSeAWhBGwGbtHMHfEs4depRQ
  (+72.070077 USDC, 2026-08-02T06:07:00Z).
- I never captured the vanished row's `payment_id` before it disappeared, so the match to this transaction is by amount and approximate date, not a direct database read. I consider it solid (exact USDC amount to the cent, right day, and the row's own "pending" status already matched the never-completed pattern before it vanished) but it is inference, not a confirmed payment_id lookup.
- A second payment on the same account is currently in the identical state: `payment_id 019f9bbc-893a-e8cb-76b6-edf9ac5adff1`, `create_time` 2026-07-26, `payment_time` 2026-08-02T07:18:18Z, `token_amount` 47.800884913, `completed` false, `tx_hash` empty. Its 30-day mark falls around 2026-08-25; I expect it to disappear the same way unless something completes it first.
- Points are unaffected by any of this: point crediting (`applyPayoutPoints`, `model/account_payment_model_plan.go:956`) runs at payment-plan creation time, independent of whether the resulting payment ever completes or later gets canceled. My account's point total kept climbing (514792.8 on 2026-08-05, 530787.9 on 2026-08-13) across the exact window the $72.07 payment disappeared.

## Affected code (main @ `6af7e029`)

- `model/account_payment_model.go:698` `CancelHungAccountPayments` — cancels on `create_time` age alone; only logs ("audit the external transfer for double payout") when `payment_record` was set, does not check it.
- `model/account_payment_model.go:787` `GetNetworkPayments` — `WHERE canceled = false`, so canceled rows are invisible to every client, including this one.
- `controller/account_payment_controller.go:217` `advancePayment` — while `payment_record` is set, Circle status `SENT`/`CONFIRMED` just returns and gets rescheduled indefinitely (`AdvancePaymentPost` reschedules until `Complete` or `Canceled`). There is no on-chain verification and no horizon after which the loop gives up or escalates; it can poll forever, and 30 days later the unrelated hung-payment sweep cancels the row out from under it. Its `CANCELLED` status arm (:269) also cancels payments, but that path would mean the processor marked a transfer that demonstrably landed on-chain as cancelled; the evidence here points to the hung sweep, not a processor cancellation.
- `model/account_payment_model.go:608` `CompletePayment` — guards `NOT canceled`, so once the sweep cancels a payment, a late Circle confirmation can no longer complete it.

## Why it matters

The account payments endpoint is supposed to be the record of what happened to a provider's payouts. Right now a payment that executed successfully on-chain can be canceled by an unrelated timeout and removed from that record with no trace: no canceled entry, no email, no change in status the client can observe, just a total that is now smaller. Anyone reconciling their own accounting against the app has no way to know the drop happened, let alone why. Of the six on-chain transfers in the batch I checked, two (my $72.07 and the still-live $47.80) never reached `completed` on the server side despite landing on-chain, so this isn't a one-off timing fluke on my account.

There is also a re-plan risk on top of the disappearance: the payout planner re-selects sweeps whose payment is canceled (`model/account_payment_model_test.go` documents this for the canceled arm of the needs-repayment query), which is exactly the "double payout" scenario `CancelHungAccountPayments`'s own log line warns about when `payment_record` was set. I don't have direct evidence of a re-plan happening for this specific payment yet: the new pending row that appeared on 2026-08-09 (`019fe526`, 224.85 GB) is not it, since its byte count and min_sweep_time don't match the 253.27 GB row and it was created before the cancellation happened. I'm flagging the re-plan as a live consequence of the current design, not something I've confirmed occurred.

I have also heard the identical symptom (a payment disappearing from the list after it appeared paid) from at least two other providers; this issue is filed from the one account measured above, and those reports are not included as evidence here.

## Suggested fix

Primary: don't auto-cancel a payment that has `payment_record` set without reconciling it first — check the processor and/or the chain for a completion that the server missed before cancelling.

Secondary: give `advancePayment` a bound. If `SENT`/`CONFIRMED` persists past some horizon, verify against the chain (or query Circle harder) instead of polling indefinitely with no escalation; and stop re-selecting sweeps whose canceled payment had `payment_record` set.

Tertiary: whatever a maintainer decides is the right ledger behavior, don't let cancellation happen silently: either keep canceled rows visible with a `canceled` status, or notify when a payment with `payment_record` set gets swept.

> [!IMPORTANT]
> I'm not picking the fix here, since it's a judgment call on the double-pay tradeoff: today's auto-cancel keeps the planner unstuck but can silently drop payments that actually landed; reconcile-before-cancel avoids that but leaves a payment stuck if reconciliation can't run for some reason.
