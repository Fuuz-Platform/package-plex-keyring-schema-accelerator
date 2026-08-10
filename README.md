# package-plex-keyring-schema-accelerator

**Version:** 0.0.1
**Dictionary:** Plex developer portal, 2026-08-05

---

## Overview

Installs the **complete Plex read surface as Fuuz data models** — 182 models,
4,936 fields — generated from Plex's own API dictionary rather than hand-written.

This is the schema layer of the Plex keyring: somewhere to land every readable Plex object, with
foreign keys resolved into real relations so applications traverse rather than join.

Models only. Connections, flows and the endpoint switchboard are separate concerns.

---

## Package Contents

```
plex-keyring-schema/
├── manifest.json
├── package-data.json
├── preinstall/     182 steps (uniqueness verification, one per model)
└── install/        911 steps (headers and versions, then relations)
```

---

## What gets installed

| | |
|---|---|
| models | **182** — one per distinct Plex response schema |
| fields | **4,936** |
| forward relations | **496** (child → parent, plus `sourceSystem` on every model) |
| reverse collections | **314** (parent → children) |

Coverage: all 119 collection GETs in the portal that resolve to a typed schema, plus the
63 schemas reachable only through a single-entity GET.

By category: masterOrTransactional 99, subResource 63, analytics 19, configuration 1.

### Largest models

| Model | Fields | Source endpoint |
|---|---|---|
| `plexUserAccount` | 81 | `/mdm/v1/user-accounts` |
| `plexShipper` | 73 | `/shipping/v1/customer-shippers` |
| `plexPart` | 72 | `/mdm/v2/parts` |
| `plexAccountsReceivableInvoice` | 71 | `/accounting/v1/ar-invoices` |
| `plexAccountsPayableInvoice` | 70 | `/accounting/v1/ap-invoices` |
| `plexCustomerAddress` | 66 | `/mdm/v1/customers/{customerId}/addresses/{addressId}` |
| `plexPurchaseOrder` | 63 | `/purchasing/v1/purchase-orders` |
| `plexCustomer` | 62 | `/mdm/v1/customers` |

---

## Three design decisions worth knowing

**Every field is nullable.** A vendor's `required` flag describes their *write* contract, not what
their read endpoint returns — optional modules, permissions and older records all produce absent
fields. One NOT NULL on a field a given tenant does not populate fails the whole upsert batch and
takes the other 499 rows with it. Only `id` is required, because the keyring computes it.

**A foreign key is three columns, not one.** Plex sends `partId` holding *their* key, while the
model's primary key is namespaced (`<sourceSystem>:<externalId>`) so two Plex instances feeding one
tenant cannot collide. Each reference therefore keeps the vendor's value as a String, adds a
computed `partRefId: ID`, and hangs the relation off that. Applications get `row.part { … }`, and
the raw vendor value is still there for tracing.

**References with nothing behind them stay plain.** Heat number, currency code, DUNS and tracking
number look like foreign keys and are not — Plex exposes no endpoint for them. Inventing a model to
point at would be a lie, so they remain strings.

---

## Provenance on every row

`externalId`, `sourceSystemId`, `sourceUrl`, `sourceVersion`, `syncedAt`, `rawPayload`, `dedupeKey`.

`sourceSystemId` is a real relation to `ExternalSystem`, not a bare id — so a mistyped source system
is rejected on the first row rather than discovered after a million have landed under a name nobody
recognises. **Create the `ExternalSystem` row before loading data**, or every insert is refused.

---

## Install order

Two passes, and pass 1 deliberately carries **no relations at all**.

A reverse collection needs its child model to exist; a child's forward relation needs its parent. No
single ordering satisfies both. Pass 1 therefore depends on nothing — the FK columns land carrying
no constraint — and pass 2 raises every model to a second version with its relations once all
182 exist. Install order cannot be wrong.

---

## Regenerating

The package is generated, never hand-edited:

```sh
node platform/gen-models.js plex     # vendor dictionary -> models
node platform/gen-package.js         # models -> this package
```

The scraped Plex dictionary is deliberately **not** shipped here. A package carries the derived Fuuz
model definitions — our own artifacts — not a redistribution of the vendor's API schemas.
