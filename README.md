# package-plex-keyring-schema-accelerator

**Version:** 0.6.0 · **Spec:** 2.0.0 · **Platform:** 2026.8.0
**Dictionary:** Plex developer portal, 2026-08-05

---

## Overview

Installs the **complete Plex read surface as Fuuz data models** — 182 models,
4,560 fields — generated from Plex's own API dictionary rather than hand-written.

This is the schema layer of the Plex keyring: somewhere to land every readable Plex object, with
foreign keys resolved into real relations so applications traverse rather than join.

Models only. Connections, flows and the endpoint switchboard are separate concerns.

---

## Package Contents

```
plex-keyring-schema-0.6.0.tgz    <- import this
  ├── manifest.json        name, version, spec + platform version
  ├── definition.json      the PackageDefinition / Version / Selections
  └── package-data.json    182 dataModels (header, version, modelDefinition, migrations)
```

The same three files are also committed unpacked, so the schema is readable and diffable in the
browser without downloading anything.

This is the layout `exportPackageArchive` produces — declarative model definitions, not a script of
create-and-deploy steps, so the order models are *installed* in is the installer's concern.

That is a different thing from the order DATA is loaded in, which very much is your concern — see
[Load order](#load-order-is-a-correctness-requirement) below.

---

## What gets installed

| | |
|---|---|
| models | **182** — one per distinct Plex response schema |
| fields | **4,560** |
| forward relations | **465** (child → parent, plus `sourceSystem` on every model) |
| reverse collections | **283** (parent → children) |

Coverage: all 119 collection GETs in the portal that resolve to a typed schema, plus the
63 schemas reachable only through a single-entity GET.

By category: masterOrTransactional 99, subResource 63, analytics 19, configuration 1.

### Largest models

Vendor fields only — relations and provenance columns excluded, so this reflects how much Plex
actually returns rather than how well connected the model is.

| Model | Vendor fields | Source endpoint |
|---|---|---|
| `plexCustomerAddress` | 50 | `/mdm/v1/customers/{customerId}/addresses/{addressId}` |
| `plexAccountsPayableInvoice` | 47 | `/accounting/v1/ap-invoices` |
| `plexShipper` | 47 | `/shipping/v1/customer-shippers` |
| `plexAccountsReceivableInvoice` | 46 | `/accounting/v1/ar-invoices` |
| `plexTruck` | 39 | `/shipping/v1/customer-trucks` |
| `plexPurchaseOrder` | 38 | `/purchasing/v1/purchase-orders` |
| `plexActivity` | 37 | `/management/v1/activity-manager/{id}` |
| `plexContainer` | 34 | `/inventory-tracking/v1/containers/{serialNumber}` |

---

## One tenant per PCN

**Install this once per Plex PCN, into its own Fuuz tenant.** The keyring is not designed for one
tenant to read several PCNs: two facilities both have a `PART-1000`, and sharing a keyspace
collapses them onto one row with no error and nothing in the data to show it happened. A tenant per
PCN makes that impossible by construction rather than by convention.

The PCN and source system are therefore tenant configuration, not request parameters — the read
flows refuse to run until both are set.

---

## Identity: the primary key IS the vendor's key

| | |
|---|---|
| `externalId` | **Plex's own key**, verbatim — `partId`, `orderNo`, whatever they call it. |
| `id` | **the same value**, copied by a create trigger. |

This looks redundant and is the single most load-bearing decision in the package, because **the
platform enforces a foreign key only when it targets `id`**. Pointed at any other unique column, a
reference resolves when it can and silently returns `null` when it cannot — no error on write, no
error on delete, and `deletionReferenceBehavior` ignored. Both behaviours were verified against a
live tenant, in both directions.

So `id = externalId` is what makes all 465 relations real rather than decorative. It also dedupes
the source for free: the primary key is the vendor's identifier, so the same record cannot land
twice however many times it is read.

A trigger sets it rather than the loader, so **any** writer gets it right — your own integration
lands the same key this package's flows would.

```
"triggers": { "create": "... $isNilOrEmpty($r.id) ? $r~>|$|{\"id\": $string($r.externalId)}| : $r" }
```

**Install one PCN per tenant.** With the vendor's key as the primary key, two PCNs in one tenant
would overwrite each other row for row.

---

## Three design decisions worth knowing

**Every field is nullable.** A vendor's `required` flag describes their *write* contract, not what
their read endpoint returns — optional modules, permissions and older records all produce absent
fields. One NOT NULL on a field a given tenant does not populate fails the whole upsert batch and
takes the other 499 rows with it. The only exceptions are the two identity columns above.

**A foreign key is the vendor's own column.** Plex sends `partId`; the Part it means is the row whose
`id` is that value. The relation points `partId → Part.id` and nothing is computed. Applications get
`row.part { … }`, the vendor's key is still right there for tracing, and a reference to a row that
does not exist is **rejected on write**.

**References with nothing behind them stay plain.** Heat number, currency code, DUNS and tracking
number look like foreign keys and are not — Plex exposes no endpoint for them. Inventing a model to
point at would be a lie, so they remain strings.

---

## Load order is a correctness requirement

Because foreign keys are enforced, **an endpoint whose parent has not loaded yet is a rejected
write** — and rows go up in batches of 500, so one bad reference fails the batch. Order is not a
freshness preference any more.

`loadLayer` on each endpoint is computed from the relation graph itself, not from naming
conventions: a model sits one layer past the deepest parent it references. 7 layers, no cycles
(checked with Tarjan — a cycle would have no total order and has to announce itself rather than
silently corrupt the sequence).

Run layers in ascending order, or just run the `plexKeyringLoadAll` flow, which chains them.

---

## Bounding a read — Plex has no pagination

Worth knowing before planning a large extract: across 283 scraped contracts and 124 collection
endpoints, Plex documents **no pagination at all** — no limit/offset, no page number, no cursor, no
continuation token. `_limit` appears on two endpoints and `$top` on one, and both are *caps* that
truncate; neither can fetch page two.

Each endpoint therefore records what it actually supports, in `paging.style`:

| | | |
|---|---|---|
| `none` | 158 | no bounding mechanism — the collection arrives as it arrives |
| `window` | 21 | a created/modified date range. Not pagination, but the only real way to bound a large read, and the basis for incremental sync |
| `cap` | 3 | a documented truncating parameter |

Windows ship **off**: silently narrowing a full extract to a slice makes data look missing rather
than bounded.

---

## Provenance on every row

`externalId`, `sourceSystemId`, `sourceUrl`, `sourceVersion`, `syncedAt`, `rawPayload`, `dedupeKey`.

`sourceSystemId` is a real relation to `ExternalSystem`, not a bare id — so a mistyped source system
is rejected on the first row rather than discovered after a million have landed under a name nobody
recognises. **Create the `ExternalSystem` row before loading data**, or every insert is refused.

---

## Regenerating

Generated, never hand-edited:

```sh
node platform/gen-models.js plex     # vendor dictionary -> models
node platform/gen-package.js         # models -> this package
```

The scraped Plex dictionary is deliberately **not** shipped here.
