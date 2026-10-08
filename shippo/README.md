# Osysharp.Carriers.Shippo

US carriers — USPS, UPS, FedEx, DHL Express — and more through [Shippo](https://goshippo.com), as a delivery provider
under [Osysharp.Delivery](https://osyrin.com/templates/kits/delivery/). The US counterpart of [Osysharp.Carriers.Sendcloud](https://osyrin.com/templates/kits/sendcloud/).

- **Labels in one call** — sending an order buys the label with the shipment inline, the carrier account and service
  level named, and answers the tracking number, the carrier's tracking link and the PDF. A refusal says why.
- **Customs** — a parcel leaving the country carries its declaration (tariff number, origin, value, the certifier).
- **Returns** — a return label on the same service, turned round by Shippo.
- **Tracking** — the carriers the shop sends with are asked, and Shippo's status is read in the contract's words.

## Use it

```osy
// app.osy
use Osysharp.Carriers.Shippo@0 { egress "api.goshippo.com"; egress "shippo-delivery.s3.amazonaws.com"; }
```

```osy
new ShippoDelivery {
  ApiKey = Secret.ShippoApiKey, SenderRegion = "CA", CustomsSigner = "…",
  Sender = new DeliveryAddress { Name = "…", Street = "…", PostCode = "…", City = "…", Country = "US" },
  Options = [ new ShippoOption { Id = "usps-priority", Label = "USPS Priority", CarrierAccount = "…",
                                 ServiceLevel = "usps_priority", Carrier = "usps", Price = 899 } ],
}
```

A test key (`shippo_test_…`) buys free test labels.

## What it does not do, and why

- **Pickups** — Shippo books a pickup FOR label transactions, whose ids the delivery contract does not carry.
- **Webhook events** — Shippo's specification declares no signature, so an event could not be told from a forgery;
  the shop asks `Track` instead (its six-hourly carrier poll).

## How it is built

`model/api/` is Shippo's own API, **generated with `osy import-api`** from its published OpenAPI spec
(`https://docs.goshippo.com/spec/shippoapi/public-api.yaml`), one namespace per area and ordinary source from then on.
Two lines were corrected by hand: the `Authorization` header is `ShippoToken <key>` as the spec's `x-token-prefix` says
(the generator is being fixed to emit that), and the two clients are named `TransactionsClient` and `TrackingClient`.
Checked against a stubbed Shippo, not a live account.
