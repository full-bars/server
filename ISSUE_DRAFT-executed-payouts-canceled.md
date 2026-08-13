# Issue draft: urnetwork/server (FINAL v3 — real links + CAUTION callout)

Title: Hung-payment canceler removes payments whose transfer already landed on-chain, with no signal to the client

## Summary

My provider account had a $72.07 payout visible as pending on 2026-08-09 and gone from `/account/payments` by 2026-08-13, even though the transfer landed on-chain at 2026-08-02T06:07:00Z (USDC +72.070077, tx below). The server never recorded completion: `tx_hash` stayed empty and `completed` stayed false. `CancelHungAccountPayments` (`model/account_payment_model.go:698`) cancels any payment where `NOT completed AND NOT canceled AND create_time < now()-30d` without checking whether the transfer's `payment_record` was set or whether the money moved. `GetNetworkPayments` (`model/account_payment_model.go:787`) filters `canceled = false`, so once the sweep fires the row disappears from the API entirely and the account total silently drops by that amount.

> [!NOTE]
> This is new code: both `CancelHungAccountPayments` and its 24h self-rescheduling task were added together in commit [bb4d0676](https://github.com/urnetwork/server/commit/bb4d067692e014d2d787a4890b0dfdd982f8e83b) (2026-07-12); the function did not exist before it. The sweep has only ever run since mid-July 2026, so the first wave of 30-day-old hung payments is only now hitting it.

## Reproduction

Account wallet: `BXqg85kyR4iMJjJwoPGZWfoPtdmoTTDE22drdmYPiLH8` (Solana, USDC payouts).

- 2026-08-09: 71 payments, including a pending $72.07 / 253.27 GB / 29,537 points
  row. It was marked Pending even though the transfer had already gone out
  on-chain (transaction below): the payment was real and landed, the pending
  marker was stale.
- 2026-08-13: 70 payments. That row is gone; no canceled row appears in its place.
- On-chain, the same wallet shows the transfer completed:
  https://solscan.io/tx/UAQjPZHhpVUJTjqgSct3AfNZwfoRFB71jmjjz3rfV4LPVywDz5jeoEjVwxXTazxfSeAWhBGwGbtHMHfEs4depRQ
  (+72.070077 USDC, 2026-08-02T06:07:00Z).
- I never captured the vanished row's `payment_id`, so the match is by exact
  amount and date, not a database lookup. Solid (exact to the cent, right day),
  but inference.
- A second payment is in the identical state today: `019f9bbc-893a-e8cb-76b6-edf9ac5adff1`, `create_time` 2026-07-26, `payment_time` 2026-08-02T07:18:18Z, `token_amount` 47.80, `completed` false, `tx_hash` empty. Its 30-day mark is around 2026-08-25; I expect it to disappear the same way unless something completes it first.
- Points are not tied to a payment's fate: crediting (`applyPayoutPoints`, `model/account_payment_model_plan.go:956`) runs at plan-creation time. My point total kept climbing (514792.8 -> 530787.9) across the window the payment disappeared, while the lifetime total earned went backward by exactly that payment ($1619.87 -> $1547.80).

## Affected code (main @ [6af7e029](https://github.com/urnetwork/server/commit/6af7e029effa08fee488c80cb5d20c7291b22863))

- `model/account_payment_model.go:698` `CancelHungAccountPayments` — cancels on `create_time` age alone; only logs ("audit the external transfer for double payout") when `payment_record` was set.
- `model/account_payment_model.go:787` `GetNetworkPayments` — `WHERE canceled = false`, so canceled rows are invisible to every client.
- `controller/account_payment_controller.go:217` `advancePayment` — while `payment_record` is set, Circle status `SENT`/`CONFIRMED` just reschedules forever; no on-chain verification, no horizon, no escalation. Its `CANCELLED` arm (:269) also cancels, but that would require the processor to mark a transfer that demonstrably landed as cancelled; the evidence points to the hung sweep.
- `model/account_payment_model.go:608` `CompletePayment` — guards `NOT canceled`, so a late confirmation after the sweep can no longer complete the payment.

## Why it matters

The payments endpoint is supposed to be the record of a provider's payouts. A payment that executed on-chain can be canceled by an unrelated timeout and removed with no trace: no canceled entry, no email, no observable status change, just a lifetime total that moved backward ($1619.87 -> $1547.80) while the points ledger kept climbing. There is no way to know the drop happened, let alone why. This is not a one-off: two of six transfers in one batch never reached `completed` on the server despite landing on-chain, and the same issue has been reported by other providers in the URnetwork support Discord.

I have no direct evidence of a re-plan yet: the new 2026-08-09 row (`019fe526`, 224.85 GB) is not one, its bytes don't match the 253.27 GB row and it predates the cancellation.

> [!CAUTION]
> Beyond the disappearing rows, this is a double-payment risk. The payout planner re-selects sweeps whose payment is canceled (the canceled arm documented in `model/account_payment_model_test.go`), and this canceler can fire after the transfer already landed, so the same bytes can be paid twice. That is exactly the case `CancelHungAccountPayments`'s own log line warns about ("audit the external transfer for double payout"). I have no direct evidence it has happened on my account, but the design permits it.

## Suggested fix

Primary: don't auto-cancel a payment with `payment_record` set without reconciling first — check the processor and/or the chain for a missed completion before cancelling.

Secondary: bound `advancePayment` — after some horizon in `SENT`/`CONFIRMED`, verify against the chain instead of polling forever; and stop re-selecting sweeps whose canceled payment had `payment_record` set.

Tertiary: don't let cancellation happen silently — keep canceled rows visible with a `canceled` status, or notify when a payment with `payment_record` set gets swept.

> [!IMPORTANT]
> I'm not picking the fix: it's a judgment call on the double-pay tradeoff. Auto-cancel keeps the planner unstuck but can silently drop payments that landed; reconcile-before-cancel avoids that but leaves a payment stuck if reconciliation can't run.
