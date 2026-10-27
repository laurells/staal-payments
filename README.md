# staal-payments

Refunds through Stripe, Paystack and Flutterwave: a [Staal](https://github.com/laurells/staal)
package that gives programs the `payments.stripe.refund`, `payments.paystack.refund` and
`payments.flutterwave.refund` capabilities, built on Staal's general `net.http`
capability. It talks to [Stripe](https://stripe.com), [Paystack](https://paystack.com)
and [Flutterwave](https://flutterwave.com) (API v3).

## Authority

This is everything the package can reach. `staalc fetch` prints it when the
package is added, and the compiler refuses the package if its code reaches
beyond it.

| Capability | Targets |
| --- | --- |
| `net.http.get` | `https://api.stripe.com/v1/payment_intents/*`, `https://api.paystack.co/transaction/verify/*`, `https://api.flutterwave.com/v3/transactions/*/verify` |
| `net.http.post` | `https://api.stripe.com/v1/refunds`, `https://api.paystack.co/refund`, `https://api.flutterwave.com/v3/transactions/*/refund` |

## Install

Add the package to your project's `staal.json`, pinned to an exact commit:

```json
{
  "dependencies": {
    "payments": {
      "git": "https://github.com/laurells/staal-payments",
      "commit": "<full 40-character commit id>"
    }
  }
}
```

Then fetch it. This writes `staal.lock` with the SHA-256 of every file, and every
later check verifies the stored copy against it:

```
staalc fetch
```

## Use

The capabilities' actions:

```
payments.stripe: refund(payment_intent: Text @target, amount: Money @amount) -> Text
payments.paystack: refund(reference: Text @target, amount: Money @amount) -> Text
payments.flutterwave: refund(transaction: Text @target, amount: Money @amount) -> Text
```

A program imports the module and asks for the capability with its own limits:

```
import payments.stripe

fn refund(intent: Text, refunds: cap payments.stripe.refund(max: 200 USD)) -> Receipt {
    return refunds.refund(intent, 25.00 USD)
}
```

The operator supplies what the library needs in the run's `grants`:

```json
{
  "grants": {
    "refunds": {
      "cap": "payments.stripe.refund",
      "needs": {
        "get": {
          "cap": "net.http.get"
        },
        "post": {
          "cap": "net.http.post",
          "idempotency_key": true
        },
        "key": {
          "env": "STRIPE_SECRET_KEY"
        }
      }
    }
  }
}
```

Paystack and Flutterwave take the same `needs` (`get`, `post`, `key`), with `post`
granted without `idempotency_key`:

```json
{
  "grants": {
    "refunds": {
      "cap": "payments.paystack.refund",
      "needs": {
        "get": { "cap": "net.http.get" },
        "post": { "cap": "net.http.post" },
        "key": { "env": "PAYSTACK_SECRET_KEY" }
      }
    }
  }
}
```

## Notes

The runtime checks the amount (`max`, `each`) and the payment intent (`only`) against the program's limits before any request is made.

Grant `post` with `"idempotency_key": true`: Stripe deduplicates refunds by Idempotency-Key, so a run resumed after a crash cannot refund twice.

Before refunding, the library reads the payment intent and refuses if its currency differs from the refund's.

Amounts are sent to Stripe in the currency's ISO 4217 minor unit. Stripe differs from ISO for a few currencies; check Stripe's currency list before refunding outside the common two-decimal currencies.

Paystack: the refund is checked against the transaction (by reference) first: it must have succeeded and be in the refund's currency. Amounts are sent in the subunit (kobo for NGN). Partial refunds are allowed, up to what is left of the transaction.

Flutterwave: the target is the transaction's numeric id (`data.id`). The refund is checked against the transaction first, as for Paystack. Flutterwave v3 takes main units (naira), so this package refunds whole units of two-decimal currencies only and refuses anything else rather than round it. Flutterwave refuses refunds below its minimum (NGN 100, KES 10).

Neither Paystack nor Flutterwave documents an idempotency key for refunds. Grant their `post` without `idempotency_key`: a refund whose answer a crash cut off stops the run in doubt, for an operator to check in the dashboard and resolve with `staalc resolve`, instead of being sent twice.

## Status

Tested end to end against a local server that answers like the service, as part of
the [Staal test suite](https://github.com/laurells/staal/blob/main/tests/packages_test.rs).
Not yet run against the live services.

## License

Copyright 2026 Laurels Ozy Echichinwo. Licensed under the
[Apache License, Version 2.0, with the LLVM Exceptions](LICENSE)
(`Apache-2.0 WITH LLVM-exception`), the same terms as
[Staal](https://github.com/laurells/staal). The licence grants no right to the
name "Staal".
