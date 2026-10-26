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

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Shipping;
using Osysharp.Http;
using Osysharp.Carriers.Sendcloud;

[Test]
void it_offers_the_shops_options_at_its_prices_where_they_go() {
  var options = Carrier().Quote(new DeliveryQuoteRequest { Currency = "EUR", Subtotal = 5000, To = new DeliveryAddress { Country = "de" } });
  Assert.Equal(["dhl"], options.Select(o => o.Id).ToList());
  Assert.Equal(595, options[0].Price);
  Assert.Empty(Carrier().Quote(new DeliveryQuoteRequest { To = new DeliveryAddress { Country = "US" } }));
}
```

From [`tests/sendcloud.test.osy`](tests/sendcloud.test.osy).

```osy
using Osysharp.Shipping;
using Osysharp.Http;
using Osysharp.Carriers.Sendcloud;

[Test]
void shipping_announces_the_parcel_with_basic_auth_and_answers_its_number_link_and_label() {
  var urls = new List<string>();
  var bodies = new List<string>();
  Http.Post.Stub((url, body, headers) => {
    urls.Add(url); bodies.Add(body);
    Assert.Equal("Basic " + Convert.ToBase64String(Encoding.UTF8.GetBytes("pub:sec")), headers["Authorization"]);
    return new HttpResponse { StatusCode = 201, Body = Announced("3SABC123", "ANNOUNCED") };
  });
  var shipment = Carrier().Ship(new ShipmentRequest { Reference = "AV-1042", OptionId = "dhl", WeightGrams = 900,
    To = new DeliveryAddress { Name = "Maja Berg", Street = "Hauptstraße 1", PostCode = "10115", City = "Berlin", Country = "DE", Email = "maja@example.test" } });
  Assert.Equal("https://panel.sendcloud.sc/api/v3/shipments/announce", urls[0]);
  Assert.Contains("\"shipping_option_code\":\"dhl_de:parcel/home\"", bodies[0]);
  Assert.Contains("\"order_number\":\"AV-1042\"", bodies[0]);
  Assert.Contains("\"address_line_1\":\"Hauptstra\\u00DFe 1\"", bodies[0]);
  Assert.Contains("\"weight\":{\"value\":\"0.900\",\"unit\":\"kg\"}", bodies[0]);
  Assert.DoesNotContain("customs_information", bodies[0]);
  Assert.Equal("3SABC123", shipment.TrackingNumber);
  Assert.Equal("https://tracking.example/3SABC123", shipment.TrackingUrl);
  Assert.Equal("%PDF-1", Text.FromBytes(shipment.Label!));
}
```

From [`tests/sendcloud.test.osy`](tests/sendcloud.test.osy).

```osy
using Osysharp.Shipping;
using Osysharp.Http;
using Osysharp.Carriers.Sendcloud;

[Test]
void a_parcel_leaving_the_eu_carries_its_customs_lines_and_postage() {
  var bodies = new List<string>();
  Http.Post.Stub((url, body) => { bodies.Add(body); return new HttpResponse { StatusCode = 201, Body = Announced("3SNO1", "ANNOUNCED") }; });
  Carrier().Ship(new ShipmentRequest { Reference = "AV-2001", OptionId = "dhl", WeightGrams = 900,
    To = new DeliveryAddress { Name = "Ola Nordmann", Street = "Karl Johans gate 1", PostCode = "0154", City = "Oslo", Country = "NO" },
    Customs = [new CustomsLine { Description = "Leather shoes", Quantity = 2, WeightGrams = 900, Value = 192m, Currency = "EUR", HsCode = "640399", OriginCountry = "IT" }],
    ExporterId = "NL123456789", Postage = 12m });
  Assert.Contains("\"customs_information\":{\"invoice_number\":\"AV-2001\",\"export_reason\":\"commercial_goods\"", bodies[0]);
  Assert.Contains("\"hs_code\":\"640399\"", bodies[0]);
  Assert.Contains("\"origin_country\":\"IT\"", bodies[0]);
  Assert.Contains("\"price\":{\"value\":\"96.00\",\"currency\":\"EUR\"}", bodies[0]);            // per unit
  Assert.Contains("\"freight_costs\":{\"value\":\"12.00\",\"currency\":\"EUR\"}", bodies[0]);
}
```

From [`tests/sendcloud.test.osy`](tests/sendcloud.test.osy).

## Learn more

- [Osysharp.Carriers.Sendcloud in the package catalogue](https://osysharp.com/packages/#osysharp-carriers-sendcloud)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
