# staal-payments

Refunds through Stripe: a [Staal](https://github.com/laurells/staal) package that gives
programs the `payments.stripe.refund` capability, built on Staal's general `net.http`
capability. It talks to [Stripe](https://stripe.com).

## Authority

This is everything the package can reach. `staalc fetch` prints it when the
package is added, and the compiler refuses the package if its code reaches
beyond it.

| Capability | Targets |
| --- | --- |
| `net.http.get` | `https://api.stripe.com/v1/payment_intents/*` |
| `net.http.post` | `https://api.stripe.com/v1/refunds` |

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

The capability's action:

```
payments.stripe: refund(payment_intent: Text @target, amount: Money @amount) -> Text
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

## Notes

The runtime checks the amount (`max`, `each`) and the payment intent (`only`) against the program's limits before any request is made.

Grant `post` with `"idempotency_key": true`: Stripe deduplicates refunds by Idempotency-Key, so a run resumed after a crash cannot refund twice.

Before refunding, the library reads the payment intent and refuses if its currency differs from the refund's.

Amounts are sent in the currency's ISO 4217 minor unit. Stripe differs from ISO for a few currencies; check Stripe's currency list before refunding outside the common two-decimal currencies.

## Status

Tested end to end against a local server that answers like the service, as part of
the [Staal test suite](https://github.com/laurells/staal/blob/main/tests/packages_test.rs).
Not yet run against the live service.

## License

Copyright 2026 Laurels Ozy Echichinwo. Licensed under the
[Apache License, Version 2.0, with the LLVM Exceptions](LICENSE)
(`Apache-2.0 WITH LLVM-exception`), the same terms as
[Staal](https://github.com/laurells/staal). The licence grants no right to the
name "Staal".
