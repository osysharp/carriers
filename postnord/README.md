# Osysharp.Carriers.PostNord — PostNord parcels from Osy#

## Use case

You send parcels in the Nordics with PostNord and want the shop to book them, print the label and follow the parcel
without anyone retyping an address. This package books a parcel and answers its PDF label, tracking number and
tracking link; reads where a parcel is; and finds the service points nearest an address. It also plugs straight into
any app built on the `Osysharp.Shipping` contract — the shop kit, for one — as a delivery provider.

## Install and use

```osy
app MyShop {
  use Osysharp.Carriers.PostNord@0 { egress "api2.postnord.com"; }
  use Osysharp.Shipping@0;
  use Osysharp.Http;
}
```

As a delivery provider, beside any others your app lists:

```osy
new PostNordDelivery {
  ApiKey = Secret.PostNordApiKey, CustomerNumber = "1234567890",
  Sender = new PostNordParty { Name = "Your Shop", Street = "Storgatan 1", PostCode = "11122", City = "Stockholm", Country = "SE" },
  CollectPrice = 4900, HomePrice = 6900, FreeFrom = 100000,
}
```

It offers **MyPack Collect** (service 19 — a service point the customer chooses: `PickupPoints` lists the ones
nearest their address, with distance and opening hours, and the booking names the chosen one as its delivery party;
with none chosen PostNord picks the nearest) and **MyPack Home** (service 17), at the prices you set: PostNord's API does not quote consumer prices, which
depend on your agreement. When the order goes out, `Ship` books it and answers the tracking number, the tracking link
and the label as a PDF.

The client on its own:

```osy
var postnord = new PostNordClient { ApiKey = Secret.PostNordApiKey, CustomerNumber = "1234567890" };
var label = postnord.Book(new PostNordBooking { ServiceCode = "19", Reference = "A-1041", Sender = …, Recipient = …, WeightGrams = 1200 });
var where = postnord.Track(label.ItemId);
var points = postnord.NearestServicePoints("SE", "11122", "Stockholm", "Storgatan");
```

## Age check at handover

A parcel that may only be handed to someone of an age (`ShipmentRequest.MinimumAge` — wine, say) is booked with
PostNord's **Age Check** (Ålderskontroll), additional service **`D8`**, and the age as a reference on the parcel itself,
of type **`ZAG`**. PostNord takes only 16, 18 or 20, so an age between them is booked at the next one up (17 → 18). On
the client, `PostNordBooking.AgeCheck = 20` does the same.

PostNord offers D8 only on parcels delivered in **Sweden**, with MyPack Home (17) and MyPack Collect (19), booked on a
Swedish customer number (`Z12`; for Home also a Danish one, `Z11`). An age-limited parcel it cannot check — to Norway,
say, or one needing 21 — is **refused before anything is booked**, with a sentence saying why, so the shop sends it with
a carrier that can.

From PostNord's own documents, read 2026-09-29:
- [General Descriptions (EDI) 22.7](https://pn-api-portal-files.s3-eu-west-1.amazonaws.com/General+Descriptions_22.7.pdf),
  D8 *Age Check*: "The age requirement to be allowed to sign for the shipment item must be presented as a reference on
  shipment item level. The reference type to be used is ZAG. The only allowed values with this reference type are 16,
  18 and 20."
- [Additional service codes](https://api2.postnord.com/rest/shipment/v3/edi/adnlservicecodes) — D8 "Ålderskontroll" /
  "Age check" — and their
  [combinations](https://api2.postnord.com/rest/shipment/v3/edi/servicecodes/adnlservicecodes/combinations): D8 with
  services 17 and 19 (and 18, Sweden to Sweden) for consignee country SE only.
- The item's `references` array is `item.references` in PostNord's
  [information model 1.3.2](https://api.swaggerhub.com/domains/postnord/PostNord-Information-Model/1.3.2), the schema
  the booking API ([shipment-v3-booking-sao](https://app.swaggerhub.com/apis-docs/postnord/shipment-v3-booking-sao/3.3.10)) references.


## Keys and hosts

The key comes from PostNord's developer portal and travels as the `apikey` query parameter. A test key works against
the acceptance-test host (`Test = true`, `atapi2.postnord.com` — grant that host too) and a production key against
`api2.postnord.com` only; a key used against the wrong one is refused, and the error says so.

The acceptance-test host name is the one PostNord's published examples use.

## What it does not cover

- An ID check with no age (`C4`), or one against the recipient's personal number (`24`).
- ZPL labels (the booking asks for PDF only).
- Push tracking events: this API is asked (`Track`), not pushed.

## Source and tests

Everything is in `model/`, with no C# anywhere. `tests/` is the producer's own project, every PostNord call
stubbed with the requests and answers PostNord's documents describe. It does not ship with the package.
