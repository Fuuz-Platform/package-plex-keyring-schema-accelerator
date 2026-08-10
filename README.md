# package-plex-keyring-schema-accelerator

**Version:** 0.2.0 · **Spec:** 2.0.0 · **Platform:** 2026.8.0
**Dictionary:** Plex developer portal, 2026-08-05

---

## Overview

Installs the **complete Plex read surface as Fuuz data models** — 182 models,
4,622 fields — generated from Plex's own API dictionary rather than hand-written.

This is the schema layer of the Plex keyring: somewhere to land every readable Plex object, with
foreign keys resolved into real relations so applications traverse rather than join.

Models only. Connections, flows and the endpoint switchboard are separate concerns.

---

## Package Contents

```
plex-keyring-schema-0.2.0.tgz    <- import this
  ├── manifest.json        name, version, spec + platform version
  ├── definition.json      the PackageDefinition / Version / Selections
  └── package-data.json    182 dataModels (header, version, modelDefinition, migrations)
```

The same three files are also committed unpacked, so the schema is readable and diffable in the
browser without downloading anything.

This is the layout `exportPackageArchive` produces — declarative model definitions, not a script of
create-and-deploy steps. Install order, relation ordering and deployment are the installer's concern.

---

## What gets installed

| | |
|---|---|
| models | **182** — one per distinct Plex response schema |
| fields | **4,622** |
| forward relations | **496** (child → parent, plus `sourceSystem` on every model) |
| reverse collections | **314** (parent → children) |

Coverage: all 119 collection GETs in the portal that resolve to a typed schema, plus the
63 schemas reachable only through a single-entity GET.

By category: masterOrTransactional 99, subResource 63, analytics 19, configuration 1.

### Largest models

| Model | Fields | Source endpoint |
|---|---|---|
| `plexUserAccount` | 81 | `/mdm/v1/user-accounts` |
| `plexPart` | 69 | `/mdm/v2/parts` |
| `plexShipper` | 66 | `/shipping/v1/customer-shippers` |
| `plexCustomerAddress` | 64 | `/mdm/v1/customers/{customerId}/addresses/{addressId}` |
| `plexAccountsPayableInvoice` | 63 | `/accounting/v1/ap-invoices` |
| `plexAccountsReceivableInvoice` | 63 | `/accounting/v1/ar-invoices` |
| `plexCustomer` | 58 | `/mdm/v1/customers` |
| `plexPurchaseOrder` | 57 | `/purchasing/v1/purchase-orders` |

---

## One tenant per PCN

**Install this once per Plex PCN, into its own Fuuz tenant.** The keyring is not designed for one
tenant to read several PCNs: two facilities both have a `PART-1000`, and sharing a keyspace
collapses them onto one row with no error and nothing in the data to show it happened. A tenant per
PCN makes that impossible by construction rather than by convention.

The PCN and source system are therefore tenant configuration, not request parameters — the read
flows refuse to run until both are set.

---

## Two identities per row, and they do different jobs

| | |
|---|---|
| `id` | **the platform's cuid.** Not derived from anything Plex sent. |
| `externalId` | **Plex's own key**, verbatim — `partId`, `orderNo`, whatever they call it. Required, unique. |

`externalId` does three jobs: it is the upsert key (so a re-extract overwrites rather than
duplicates), it is what every foreign key in the module resolves against, and it is the tie back to
the record being mirrored. `id` does one: identify the row, forever, regardless of what Plex later
does to its own numbering.

Keeping them separate is what makes both safe. A primary key derived from vendor data is a natural
key wearing a primary key's clothes — the day Plex renumbers or re-keys anything, the identity of a
row that other rows already point at changes underneath them. And a cuid cannot collide across PCNs
no matter how the tenants are arranged, which a namespaced key only achieves by convention.

---

## Three design decisions worth knowing

**Every field is nullable.** A vendor's `required` flag describes their *write* contract, not what
their read endpoint returns — optional modules, permissions and older records all produce absent
fields. One NOT NULL on a field a given tenant does not populate fails the whole upsert batch and
takes the other 499 rows with it. The only exceptions are the two identity columns above — `id`,
which the platform supplies, and `externalId`, which the mapping does.

**A foreign key is the vendor's own column.** Plex sends `partId`; the Part it means is the row
whose `externalId` is that value. So the relation points `partId → Part.externalId` and nothing is
computed. Applications get `row.part { … }` and the vendor's key is still right there for tracing.

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

## Regenerating

Generated, never hand-edited:

```sh
node platform/gen-models.js plex     # vendor dictionary -> models
node platform/gen-package.js         # models -> this package
```

The scraped Plex dictionary is deliberately **not** shipped here.
