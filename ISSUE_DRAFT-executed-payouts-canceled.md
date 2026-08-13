# Issue draft: urnetwork/server (v4, account-attributed, with tx link)

Title: Executed payouts are auto-canceled as "hung" and silently vanish from /account/payments

## Summary

The 30-day hung-payment canceler does not distinguish a payment whose transfer
was never submitted from one whose transfer was submitted and landed. Any pending
payment older than 30 days from create_time is canceled, and the account payments
list filters canceled rows out entirely. A provider whose payout transferred
successfully on-chain but never flipped to completed therefore sees the payment
disappear from the app with no status, no notification, and no row in the API
response, while the lifetime total drops by that payment's amount.

All measurements below are from a single provider account, the reporter's own:
wallet `BXqg85kyR4iMJjJwoPGZWfoPtdmoTTDE22drdmYPiLH8` (Solana, USDC). The account
showed 71 payments on 2026-08-09 and 70 on 2026-08-13; the missing row was
$72.07 / 253.27 GB / 29,537 points / 13,314 reliability. The wallet balance
confirms the transfer landed on-chain at 2026-08-02T06:07:00Z (USDC +72.070077,
tx `UAQjPZHhpVUJTjqgSct3AfNZwfoRFB71jmjjz3rfV4LPVywDz5jeoEjVwxXTazxfSeAWhBGwGbtHMHfEs4depRQ`).
The server's copy had token_amount and payment_time set but tx_hash empty: the
transfer executed out of band and completion was never recorded.

> [!NOTE]
> New behavior, not an old regression: the canceler model function dates to
> `urnetwork/server#231` (2025-05-22), but the task was never scheduled until
> commit `bb4d0676` "performance optimizations and fixes" (2026-07-12, shipped in
> the v2026.7.15 release). The sweep started running mid-July 2026, so providers
> are only now hitting the 30-day arm. Related prior report:
> `urnetwork/server#376`.

## Reproduction

- GET /account/payments on the reporter's account: 71 payments (2026-08-09) -> 70
  (2026-08-13); the 253.27 GB row is absent from the later response.
- The landed transfer for the missing row:
  https://explorer.solana.com/tx/UAQjPZHhpVUJTjqgSct3AfNZwfoRFB71jmjjz3rfV4LPVywDz5jeoEjVwxXTazxfSeAWhBGwGbtHMHfEs4depRQ
- A second payment on the same account is in the same state today: token_amount
  47.80, payment_time 2026-08-02T07:18:18Z, completed false, tx_hash empty. Its
  create_time (2026-07-26) means the canceler will remove it around 2026-08-25
  unless the completion path catches up first.
- Points are not reversed by the cancel: the account total went 514,792.8
  (2026-08-05) -> 530,787.9 (2026-08-13) across the disappearance. Only the
  payment row vanishes; the points it earned stay credited, so a provider sees
  their points keep climbing while a payment built on those points disappears.

## Affected code (main @ `6af7e029`)

- `model/account_payment_model.go:678` `CancelHungAccountPayments`: cancels every
  row WHERE NOT completed AND NOT canceled AND create_time < now()-30d; only logs
  when payment_record was set ("audit the external transfer for double payout").
  Never checks whether the transfer landed.
- `model/account_payment_model.go:805` `GetNetworkPayments`: WHERE ... canceled =
  false, so canceled rows are invisible to every client.
- `controller/account_payment_controller.go:217` `advancePayment`: completion is
  driven solely by the processor status flipping to COMPLETE; SENT/CONFIRMED is
  polled indefinitely with no on-chain fallback and no timeout.
- `model/account_payment_model.go:608` `CompletePayment`: guards NOT canceled, so
  a late confirmation after cancellation cannot complete the payment.

## Why it matters

The list API claims to return the account's payments but returns only non-canceled
ones, and the canceler can fire after the money moved. There is no signal: no
canceled status, no notification, the total just drops, so operators deploy
assuming the ledger is complete when it is not. The re-plan path then re-selects
the canceled payment's sweeps, paying the same bytes a second time, the exact
double-pay case the canceler's own log comment names. Live data shows the
triggering state is not rare: two of six transfers in one batch never reached
completed.

The reporter has also heard the identical symptom (a payment disappearing from
the list after it appeared paid) from at least two other providers; this issue is
filed from the one account measured above, and those reports are not included as
evidence here.

## Suggested fix

Primary: never auto-cancel a payment with payment_record set; reconcile it
on-chain first and only cancel and re-plan if the transfer truly never landed.
Subscriptions already have a webhook reconciliation controller; provider payouts
have none.

Secondary: in advancePayment, treat SENT/CONFIRMED past a horizon as
complete-on-verification instead of polling indefinitely; stop re-selecting
sweeps whose canceled payment had payment_record set.

Tertiary: surface canceled payments to clients so removal is never silent.

> [!IMPORTANT]
> The tradeoff is on the double-pay side. Auto-canceling keeps the ledger moving
> but can pay the same bytes twice; reconcile-before-cancel risks a stuck payment
> if verification cannot run. Happy to send a patch in either direction,
> including removing the payment_record arm of the canceler and leaving those
> payments pending for manual review.
