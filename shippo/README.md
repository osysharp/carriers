# Osysharp.Carriers.Shippo

US carriers — USPS, UPS, FedEx, DHL Express — and more through [Shippo](https://goshippo.com), as a delivery provider
under [Osysharp.Shipping](https://osyrin.com/templates/kits/shipping/). The US counterpart of [Osysharp.Carriers.Sendcloud](https://osyrin.com/templates/kits/sendcloud/).

- **Labels in one call** — sending an order buys the label with the shipment inline, the carrier account and service
  level named, and answers the tracking number, the carrier's tracking link and the PDF. A refusal says why.
- **Customs** — a parcel leaving the country carries its declaration (tariff number, origin, value, the certifier).
- **Returns** — a return label on the same service, turned round by Shippo.
- **Tracking** — the carriers the shop sends with are asked, and Shippo's status is read in the contract's words.

## Use it

```osy
// app.osy
use Osysharp.Carriers.Shippo@1 { egress "api.goshippo.com"; egress "shippo-delivery.s3.amazonaws.com"; }
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
Two lines are set by hand: the `Authorization` header is `ShippoToken <key>` as the spec's `x-token-prefix` says, and
the two clients are named `TransactionsClient` and `TrackingClient`. The tests run against a stubbed Shippo that answers
as the published spec describes.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Shipping;
using Osysharp.Http;
using Osysharp.Carriers.Shippo;

[Test]
void it_offers_the_shops_options_where_they_go() {
  Assert.Equal(["usps-priority"], Carrier().Quote(new DeliveryQuoteRequest { To = new DeliveryAddress { Country = "us" } }).Select(o => o.Id).ToList());
  Assert.Empty(Carrier().Quote(new DeliveryQuoteRequest { To = new DeliveryAddress { Country = "SE" } }));
}
```

From [`tests/shippo.test.osy`](tests/shippo.test.osy).

```osy
using Osysharp.Shipping;
using Osysharp.Http;
using Osysharp.Carriers.Shippo;

[Test]
void shipping_buys_the_label_in_one_call_and_answers_number_link_and_pdf() {
  var bodies = new List<string>();
  Http.Post.Stub((url, body, headers) => {
    Assert.Equal("https://api.goshippo.com/transactions", url);
    Assert.Equal("ShippoToken shippo_test_k", headers["Authorization"]);
    bodies.Add(body);
    return new HttpResponse { StatusCode = 201, Body = Bought("SUCCESS") };
  });
  Http.Get.Stub(url => new HttpResponse { StatusCode = 200, Body = "%PDF-1", Bytes = Text.ToBytes("%PDF-1") });
  var s = Carrier().Ship(new ShipmentRequest { Reference = "RC-1042", OptionId = "usps-priority", WeightGrams = 900,
    To = new DeliveryAddress { Name = "Ann Lee", Street = "5 Main St", PostCode = "10001", City = "New York", Country = "US" } });
  Assert.Contains("\"carrier_account\":\"ca_1\"", bodies[0]);
  Assert.Contains("\"servicelevel_token\":\"usps_priority\"", bodies[0]);
  Assert.Contains("\"label_file_type\":\"PDF\"", bodies[0]);
  Assert.Contains("\"address_from\":{\"name\":\"Shop\",\"street1\":\"1 Market St\",\"city\":\"San Francisco\",\"state\":\"CA\"", bodies[0]);
  Assert.Contains("\"weight\":\"900\",\"mass_unit\":\"g\"", bodies[0]);
  Assert.Equal("9400111", s.TrackingNumber);
  Assert.StartsWith("https://tools.usps.com", s.TrackingUrl);
  Assert.Equal("%PDF-1", Text.FromBytes(s.Label!));
}
```

From [`tests/shippo.test.osy`](tests/shippo.test.osy).

```osy
using Osysharp.Shipping;
using Osysharp.Http;
using Osysharp.Carriers.Shippo;

[Test]
void a_parcel_leaving_the_country_carries_its_customs_declaration() {
  var bodies = new List<string>();
  Http.Post.Stub((url, body) => { bodies.Add(body); return new HttpResponse { StatusCode = 201, Body = Bought("SUCCESS") }; });
  Http.Get.Stub(url => new HttpResponse { StatusCode = 200, Body = "%PDF-1", Bytes = Text.ToBytes("%PDF-1") });
  Carrier().Ship(new ShipmentRequest { Reference = "RC-2001", OptionId = "usps-priority", WeightGrams = 900,
    To = new DeliveryAddress { Name = "Jean Tremblay", Street = "1 Rue X", PostCode = "H2X 1Y4", City = "Montreal", Country = "CA" },
    Customs = [new CustomsLine { Description = "Bicycle frame", Quantity = 1, WeightGrams = 900, Value = 450m, Currency = "USD", HsCode = "871491", OriginCountry = "IT" }] });
  Assert.Contains("tariff_number", bodies[0]);
  Assert.Contains("871491", bodies[0]);
  Assert.Contains("Sam Shop", bodies[0]);                                      // who certifies it
}
```

From [`tests/shippo.test.osy`](tests/shippo.test.osy).

## Learn more

- [Osysharp.Carriers.Shippo in the package catalogue](https://osysharp.com/packages/#osysharp-carriers-shippo)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
