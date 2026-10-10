# Osysharp.Carriers.Sendcloud

Many European carriers behind one integration — DHL, DPD, GLS, UPS, PostNL, Colissimo and more — through
[Sendcloud](https://www.sendcloud.com)'s API v3. A delivery provider under [Osysharp.Shipping](https://osyrin.com/templates/kits/shipping/), so a shop
lists it beside any other way to deliver.

- **Labels** — sending an order announces the parcel and answers its tracking number, tracking link and PDF label.
  A carrier's refusal, which Sendcloud answers with 201, is read as the failure it is.
- **Customs** — a parcel leaving the EU carries its lines (tariff number, origin, value, weight) and the postage;
  Sendcloud makes the customs papers.
- **Tracking and events** — the newest `phase` in the contract's words (booked, on its way, ready to collect,
  delivered, returned, problem), asked or pushed through Sendcloud's event subscriptions.
- **Returns** — from the customer to the shop, with a label to print.
- **Pickups** — the carrier comes to the shop in the day's window.

## Use it

```osy
// app.osy
use Osysharp.Carriers.Sendcloud@0 { egress "panel.sendcloud.sc"; }
```

```osy
// model/shop.osy
new SendcloudDelivery {
  PublicKey = Secret.SendcloudPublicKey, SecretKey = Secret.SendcloudSecretKey,
  Sender = new DeliveryAddress { Name = "…", Street = "…", PostCode = "…", City = "…", Country = "NL" },
  Options = [ new SendcloudOption { Id = "dhl", Label = "DHL Parcel", Code = "dhl_de:parcel/home", Price = 595 } ],
  ReturnCode = "postnl:return", PickupCarrier = "dhl_express", EventToken = Secret.SendcloudEventToken,
}
```

`Test = true` ships every parcel as `sendcloud:letter` and every return as `sendcloud:letterreturn`, which Sendcloud
does not charge for — Sendcloud has no sandbox.

## How it is built

`model/api/` is Sendcloud's own API, **generated with `osy import-api`** from its published OpenAPI specs
(`https://sendcloud.dev/.openapi/v3/<area>/openapi.yaml`), one namespace per area — shipments, returns, pickups,
tracking, documents — and ordinary source from then on. `SendcloudDelivery` and four small adapters map the delivery
contract onto it.

⚑ **The tests run against a stubbed Sendcloud**, and every request and answer in them follows the published specs.
Two things the specs leave open are settled here: a return answers only its ids (the label is fetched from the
documents API, and the tracking number arrives with the parcel's events), and a v3 event carries no signature — it is
admitted by the bearer token configured on the connection.
