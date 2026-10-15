<pre>
  CIP: ?
  Layer: Daml
  Title: Canton Data Standard
  Author: Charles Desmonty <charles.desmonty@kaiko.com>
  Status: Draft
  Type: Standards Track
  Created: 2026-10-05
  License: CC0-1.0
  License-Code: Apache-2.0
</pre>


## Abstract

Define standard Daml interfaces for publishing and consuming data on Canton
Network, so that an application can read prices, reference rates, NAVs and any
other data point from every distributor that implements the standard, without
depending on that distributor's packages, and can switch or combine
distributors without changing its code.

The standard consists of two interfaces and two supporting packages.
`PublishedDataPoint` exposes an on-ledger publication: the distributor, the
publication time, a schema identifier and a typed key/value payload.
`DistributorKey` publishes a distributor's secp256k1 public key, so that
payloads signed off-ledger are verified inside the consumer's own transaction,
with nothing written on-ledger per payload. A data package holds the shared
value types. An upgradable utility library holds the hashing and verification
code: a structural hash that off-ledger signers reproduce, the signed
envelope, signature verification, and a codec for price quotes.

This CIP also proposes to publish these packages in Splice.


## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).
The Daml code of the reference implementation is licensed under
[Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0).


## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted
as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### Overview

The standard is concerned with two roles:

- **distributors**: parties that publish data on Canton. For example oracle
  providers, benchmark administrators, or fund administrators publishing NAVs.
- **consumers**: Daml applications whose workflows depend on that data. For
  example lending protocols, DvP and settlement apps, collateral management, or
  tokenized fund subscription and redemption workflows.

A distributor publishes by implementing a standard interface on a template.
A consumer reads through that interface and never names the distributor's package.
Integration is done once against the standard; switching distributor doesn't
require a new DAR file.

The standard supports two delivery models:

- **Push**: the distributor creates one contract per publication, implementing
  `PublishedDataPoint`.
- **Pull**: the distributor signs payloads off-ledger and hands them to
  consumers through its own channel. On-ledger, it publishes one long-lived
  contract implementing `DistributorKey`. A consumer authenticates any number
  of payloads against that key by calling the verification library from its
  own choice.

Both models carry the same payload type and the same schema identifiers, so
everything a consumer does after the read is shared between them.

The standard fixes what a consumer reads and how the consumer authenticates
it. Request flows, the transport of disclosed contracts and signed payloads,
pricing, entitlements and receipts belong to each distributor's product.

### Packages

| Package | Content | Upgrade model |
|---|---|---|
| `canton-data-standard-utils-v1` | Shared types: `AnyValue`, `Values`, `Metadata`, `SignedDataPoint`, `VerifiedDataPoint`, and the `insertField` / `lookupField` helpers | No templates or interfaces; evolves by smart-contract upgrade (SCU) |
| `canton-data-standard-datapoint-v1` | The `PublishedDataPoint` interface | Interface package: never upgraded; a breaking change is a new `-v2` package |
| `canton-data-standard-distributor-key-v1` | The `DistributorKey` interface | Interface package, as above |
| `canton-data-standard-codecs` | Structural hash, signed envelope, signature verification, `Quote` codec | Utility package built with `--force-utility-package`: any two versions are SCU-compatible |

Interface packages contain views and one read choice each.
A package that defines an interface cannot be upgraded. Hashing,
verification and codecs therefore live in the codecs package,
and `utils-v1` holds only types and small helpers. Every package
depends only on `daml-prim`, `daml-stdlib` and other packages of the
standard.

Package and module names above are those of the reference release. On
inclusion in Splice, the packages take Splice's naming conventions for API
packages (for example `splice-api-data-datapoint-v1` with module
`Splice.Api.Data.DataPointV1`). The final names are agreed with the Splice
maintainers and recorded in this CIP before it is proposed for a vote.

### Value types

```haskell
-- canton-data-standard-utils-v1, module DataStandard.Utils

data AnyValue
  = AVInt Int
  | AVDecimal Decimal
  | AVText Text
  | AVTime Time
  | AVBool Bool
  | AVList [AnyValue]
  | AVMap (TextMap AnyValue)

type Values = TextMap AnyValue

data Metadata = Metadata with
    values : TextMap Text
```

`AnyValue` is a closed set: a distributor cannot add a constructor. Lists and
maps nest, so structured payloads such as the constituents of an index or the
term structure of a curve are expressible. Adding a constructor is a versioned
change of the standard (see [Evolution](#evolution)).

`Values` maps field names to tagged values. Which fields a payload carries, and
at which tags, is defined by the schema that the publication names.

`Metadata` carries annotations that are not part of the data, such as
provenance, methodology references or links. Keys follow the metadata key
syntax of [CIP-0056](../cip-0056/cip-0056.md#metadata-key-syntax).
Consumers MUST ignore keys they do not recognize. A distributor adds an
annotation as a metadata entry rather than by changing a view.

### PublishedDataPoint

```haskell
-- canton-data-standard-datapoint-v1, module DataStandard.DataPointV1

data PublishedDataPointView = PublishedDataPointView with
    distributor   : Party
    publishedAt   : Time
    schemaVersion : Text
    values        : Values
    metadata      : Metadata

interface PublishedDataPoint where
  viewtype PublishedDataPointView

  nonconsuming choice PublishedDataPoint_Fetch : PublishedDataPointView
    with
      actor : Party
    controller actor
    do pure (view this)
```

| Field | Meaning |
|---|---|
| `distributor` | The party publishing the data, and the party a consumer decides to trust. |
| `publishedAt` | When the distributor produced the data. This can precede the creation of the contract. |
| `schemaVersion` | The schema of `values`. See [Schemas](#schemas). |
| `values` | The payload. |
| `metadata` | Annotations outside the data. |

Distributors:

- MUST implement the interface only on templates of which `distributor` is a
  signatory.
- SHOULD refresh a publication by archive-and-replace: a consuming choice that
  creates the successor. A consumer holding the previous contract id then fails
  instead of reading stale data. Revocation is archival without a successor.

Consumers reference publications as `ContractId PublishedDataPoint` and read
them through `PublishedDataPoint_Fetch`. The choice is authorized by its
`actor` alone, which makes it usable on a contract held by explicit disclosure,
where a plain `fetch` fails because the reader is not a stakeholder.

### Schemas

`schemaVersion` identifies the schema of `values`: the field names, their
`AnyValue` tags and their meaning. There are two kinds of schema identifiers:

- **Standard schemas** are defined by this CIP and its amendments. They are
  named `<name>-<major>` and mean the same for every distributor. This CIP
  defines `quote-1`.
- **Distributor schemas** are defined by a distributor for its own feeds. They
  SHOULD be semantic versions, and their meaning is scoped to the distributor:
  the pair (`distributor`, `schemaVersion`) identifies the schema. Distributors
  SHOULD publish documentation of every schema they use.

Where a schema carries a feed identifier, the distributor MUST keep it unique
among its feeds and MUST NOT reassign it to a different feed. The pair
(`distributor`, feed identifier) is the unit a consumer trusts. How a
distributor allocates its identifiers is its own concern and need not be
recorded on-ledger.

#### The `quote-1` schema

A price quote is a payload of exactly three fields:

| Field | Tag | Meaning |
|---|---|---|
| `feedId` | `AVText` | The feed, for example `BTC/USD`. Subject to the feed identifier rules above. |
| `price` | `AVDecimal` | The price, as an exact base-10 fixed-point number. |
| `priceTime` | `AVTime` | The market instant the price is observed for. Equal to `publishedAt` for a feed without a distinct observation time. |

A payload under `quote-1` MUST contain these three fields at these tags and no
other field. A feed with more to convey uses a schema of its own.

The codecs package provides `Quote`, an in-memory record of the three fields,
with total encoding (`quoteToValues`), strict decoding (`valuesToQuote`, which
rejects a missing field, a wrong tag or an extra field), and the pull-path
helpers `signedQuote` and `verifiedQuote`. Because the codecs package is a
utility package, `Quote` is not serializable: templates store the `Values`
payload.

### DistributorKey

```haskell
-- canton-data-standard-distributor-key-v1, module DataStandard.DistributorKeyV1

data DistributorKeyView = DistributorKeyView with
    distributor  : Party
    publicKey    : Text
    signMethod   : Text
    hashMethod   : Text
    payloadCodec : Text

interface DistributorKey where
  viewtype DistributorKeyView

  nonconsuming choice DistributorKey_Fetch : DistributorKeyView
    with
      actor : Party
    controller actor
    do pure (view this)
```

| Field | Meaning |
|---|---|
| `distributor` | The party whose signatures the key validates. Verification copies it into its result. |
| `publicKey` | The secp256k1 public key, hex-encoded DER SubjectPublicKeyInfo. |
| `signMethod` | `"secp256k1"` for the codec defined in this CIP. |
| `hashMethod` | `"SHA-256"` for the codec defined in this CIP. |
| `payloadCodec` | The identifier of the envelope the key signs. This CIP defines `v2-datapoint-hash`. Verification refuses a key whose codec it does not implement; `signMethod` and `hashMethod` are descriptive and not checked. |

Distributors:

- MUST implement the interface only on templates of which `distributor` is a
  signatory.
- MAY authenticate all their feeds with one key.
- SHOULD rotate a key by archive-and-replace, through a choice on their own
  template and under their own rotation policy. For consumers that read the
  key in the verifying transaction, as required below, payloads signed under
  an archived key stop verifying, because an archived contract cannot be used
  in a transaction.

Reads are non-consuming, so any number of consumers verify concurrently against
one key contract without contention.

### Signed envelope

A distributor signs a `SignedDataPoint`:

```haskell
-- canton-data-standard-utils-v1

data SignedDataPoint = SignedDataPoint with
    publishedAt   : Time
    expiresAt     : Time
    schemaVersion : Text
    values        : Values
```

`publishedAt`, `schemaVersion` and `values` have the meaning they have on
`PublishedDataPointView`. `expiresAt` is the last instant at which the payload
is accepted, and MUST NOT precede `publishedAt`. Times in a signed payload,
including `AVTime` values inside `values`, MUST be whole milliseconds at or
after the Unix epoch.

The signature travels beside the record and is computed as follows.

**Structural hash.** `H(t)` denotes the lowercase hexadecimal SHA-256 of the
UTF-8 bytes of text `t`, with no Unicode normalization. Nodes combine hashes
with the text `|`:

- a list or record of hashes `h1 ... hN` hashes as `H("N|h1|...|hN")`, with
  `N` in decimal; the empty list hashes as `H("0")`;
- a variant with tag `tag` and field hashes `h1 ... hN` hashes as
  `H("tag|N|h1|...|hN")`.

Leaves hash as `H` of their rendering:

| Type | Rendering |
|---|---|
| `Int` | Decimal, with a leading `-` for negatives: `42`, `-5` |
| `Decimal` | Plain decimal notation, no exponent, no trailing zeros, at least one fractional digit: `1.0875`, `65000.0`, `-12.5` |
| `Text` | The text itself |
| `Bool` | `true` or `false` |
| `Time` | Integer milliseconds since the Unix epoch: `1768480245000` |

An `AnyValue` hashes as a variant with one field, under these tags:

| Constructor | Tag | Field hash |
|---|---|---|
| `AVInt i` | `int` | leaf hash of `i` |
| `AVDecimal d` | `decimal` | leaf hash of `d` |
| `AVText t` | `text` | leaf hash of `t` |
| `AVTime t` | `time` | leaf hash of `t` |
| `AVBool b` | `bool` | leaf hash of `b` |
| `AVList xs` | `list` | list of the hashes of the elements, in order |
| `AVMap m` | `map` | map hash of `m` |

A map hashes as the list of its entries in ascending Unicode code point order
of their keys (which is also raw UTF-8 byte order; UTF-16 code unit order
differs for keys outside the Basic Multilingual Plane), each entry hashing as
the record `[H(key), hash(value)]`.

**Envelope.** The root hash of a `SignedDataPoint` `p` is the record:

```
root = record [ leaf(p.publishedAt), leaf(p.expiresAt), leaf(p.schemaVersion), hash(AVMap p.values) ]
```

**Signature.** The distributor signs with ECDSA on secp256k1, with SHA-256 as
the digest, over the 32 bytes obtained by hex-decoding `root`. The signature is
hex-encoded DER. The codec identifier of this envelope is `v2-datapoint-hash`.

The structural combinators are those of `Splice.Amulet.CryptoHash`; the
standard adds the `AnyValue`, `Bool`, `Time` and `TextMap` cases. An
off-ledger signer needs SHA-256, hexadecimal encoding, string joins, the
renderings above and a secp256k1 signer. The golden vectors in the reference
implementation (`tests-codecs`) are normative. Examples:

| Payload | Root hash |
|---|---|
| `publishedAt` = 1768480245000, `expiresAt` = 1768483845000, `schemaVersion` = `1.0.0`, `values` = {`feedId`: `AVText "EUR/USD"`, `rate`: `AVDecimal 1.0875`} | `1ac5fa4ba0227c3e348464699c40b5f313ffbae4c12943e544d2e44d176b575c` |
| `publishedAt` = 1768480245000, `expiresAt` = 1768483845000, `schemaVersion` = `quote-1`, `values` = {`feedId`: `AVText "BTC/USD"`, `price`: `AVDecimal 65000.0`, `priceTime`: `AVTime 1768480245000`} | `156a45a913599d7597df9e065d9f5b3dd57b41d5910da81e148368c9061de19e` |

An informative implementation of the hash, checked against the golden vectors,
is given in the [appendix](#appendix-off-ledger-hash-implementation).

### Verification

```haskell
-- canton-data-standard-utils-v1

data VerifiedDataPoint = VerifiedDataPoint with
    distributor   : Party
    publishedAt   : Time
    schemaVersion : Text
    values        : Values
    canonicalHash : Text
    signature     : Text
    publicKey     : Text

-- canton-data-standard-codecs, module DataStandard.Codecs.Verify

verifyDataPoint
  : DistributorKeyView -> Time -> SignedDataPoint -> Text
  -> Either Text VerifiedDataPoint

verifyDataPointWith
  : Party -> Text -> Time -> SignedDataPoint -> Text
  -> Either Text VerifiedDataPoint
```

`verifyDataPoint key now payload signature` runs these checks in order and
returns the first failure:

| Check | Failure |
|---|---|
| `key.payloadCodec == "v2-datapoint-hash"` | `"unsupported payload codec"` |
| `payload.publishedAt <= payload.expiresAt` | `"published after expiry"` |
| `now <= payload.expiresAt` | `"payload expired"` |
| ECDSA verification of `signature` over the root hash under `key.publicKey` | `"invalid signature"` |

On success it returns a `VerifiedDataPoint` whose `distributor` is taken from
the key, whose leading fields mirror `PublishedDataPointView`, and whose last
three fields are `canonicalHash` (the root hash), `signature` and `publicKey`.
Together with `expiresAt`, which the result does not carry and which a consumer
keeps if it needs to, they are the evidence needed to reproduce the check off-ledger.
`canonicalHash` is a stable join key for any record that refers to the verification.
There is no `metadata` field: the envelope signs none, and annotations delivered
alongside a signed payload are not authenticated.

`verifyDataPointWith` performs the last three checks against a distributor and
key the caller supplies, for callers that obtain the key by other means than a
`DistributorKey` contract. The failure texts are part of the public surface of
the codecs package.

A consumer runs verification inside its own choice, and every validating
participant re-executes it as it would an interface choice. A defect in
it is fixed by a codecs release; the interfaces and the distributors'
implementations stay unchanged.

### Consumer requirements

Consumers:

- MUST depend only on the standard's packages to read data, and reference
  publications and keys by interface.
- MUST check that `distributor` is a party they trust for the feed, that the
  payload describes the feed they expect, and that `schemaVersion` is one they
  support.
- MUST bound the age of `publishedAt` against ledger time, and SHOULD reject a
  `publishedAt` later than ledger time beyond their tolerance for clock skew.
- MUST NOT fail on `values` fields or `metadata` keys they do not use, except
  where a standard schema fixes the exact set of fields, as `quote-1` does.
- MUST, on the pull path, read the key through `DistributorKey_Fetch` in the
  verifying transaction. `expiresAt` is the standard's only replay bound;
  a consumer SHOULD apply its own, tighter freshness policy on top of it.

### Visibility

A consumer reads only contracts it can see, and the standard supports two ways
of making a publication visible:

- **Explicit disclosure.** The distributor hands the consumer the
  `template_id`, `contract_id` and `created_event_blob` off-ledger, and the
  consumer attaches them to its command submission. Storage does not grow with
  the audience, and disclosure is tamper-evident because a contract id is a
  hash of the contract's contents.
- **Observers.** Naming consumers as observers makes them stakeholders: the
  publication reaches their participant, they can query it and `fetch` it
  directly. Storage and streaming then grow with the audience, which suits
  small, known audiences.

On both paths, the distributor is a signatory of the contract a consumer
reads, and so an informee of every read: the distributor's participant
processes the reading part of each consumer transaction and learns which
party read which publication or key, and when. It does not see the rest of the
consumer's transaction.

### Evolution

Changes to the standard fall into three categories:

- **Editorial**: documentation and comment changes. They MAY be made at any
  time and need no amendment.
- **Non-breaking**: fixes and additions to the codecs package, released as
  ordinary version bumps that consumers adopt by rebuilding; new metadata
  keys; new standard schemas; new codec identifiers. New standard schemas and
  codec identifiers extend the signed surface that off-ledger implementers
  rely on, and SHOULD be recorded as amendments to this CIP.
- **Breaking**: any change to an interface view or choice, which is made as a
  new `-v2` interface package beside `-v1`, and any new `AnyValue`
  constructor, which a reader built against an older `utils-v1` cannot decode.
  Breaking changes require an amendment to this CIP or a new CIP. The `-v1`
  packages remain available.

The codecs package carries no `-v1` suffix because its versions are mutually
upgrade-compatible. The signed encoding is versioned separately, by codec
identifier.


## Motivation

Every financial application on Canton consumes data: a lending protocol needs
collateral prices, a settlement app a reference rate, a tokenized fund its NAV.
Today each application integrates each data source bespoke, against that
source's templates. Adding a second source or switching sources means new Daml
code and a new package dependency, and data providers entering the network face
one bilateral integration per application.

A shared interface makes the read side uniform. Applications integrate once,
data providers become interchangeable at the level of a party, and tools such
as indexers, dashboards and cross-checks between providers work across all
implementations.


### Use cases

The distribution models below are covered by the standard, each with its
reference implementation in the
[repository](https://github.com/kaikodata/canton-data-standard) where one
exists. The reference consumers depend only on the standard's packages.

| Model | How it uses the standard | Reference implementation |
|---|---|---|
| Push, generic payload | One contract per publication, read field by field with `lookupField` | `examples/datapoint-producer`, `examples/datapoint-consumer` |
| Push, price quote | The same interface under `quote-1`, decoded with `valuesToQuote` | `examples/quote-producer`, `examples/quote-consumer` |
| Pull, signed payloads | One `DistributorKey`; payloads signed off-ledger and verified in the consumer's choice; key rotation | `examples/distributor-key-producer`, `examples/distributor-key-consumer` |
| Distributor switching | One consumer reading two structurally different distributors (a stored price; a bid and ask with a derived mid) without code change | `examples/switching-distributor-direct`, `examples/switching-distributor-marketmaker`, `examples/switching-consumer` |
| Multi-distributor cross-check | Two distributors read in one transaction; settlement only when they agree within a tolerance | `CrossCheckOffer` in `examples/switching-consumer` |
| Request-response | The distributor answers a consumer's request by creating a `PublishedDataPoint`; the request leg is the distributor's product | Consumer side as in `examples/datapoint-consumer` |

## Rationale

### Relation to oracle interfaces on other chains

Price feed interfaces on EVM chains, such as Chainlink's `AggregatorV3Interface`
(`latestRoundData()` returning an integer answer and its update time) and
[ERC-7726](https://eips.ethereum.org/EIPS/eip-7726) (`getQuote(baseAmount,
base, quote)`), standardize a read call on a feed contract. The push path is
the Canton counterpart: `publishedAt` corresponds to the update time, the pair
(`distributor`, `feedId`) to the feed contract, and `price` to the answer,
carried as a `Decimal` instead of an integer with decimals. Signed-report oracles,
where a consumer verifies a provider's signature on-chain, correspond to the
pull path. Two differences follow from Canton's model: the consumer reads a
contract by disclosure or as a stakeholder rather than from global state, and
the payload is a typed map rather than a single number.

### One payload interface instead of a typed quote interface

The Development Fund proposal for this standard and earlier drafts defined a
separate `PublishedQuote` interface. A quote is a payload of three fields, so
it now travels the same interface and the same signed envelope as every other
payload, under `quote-1`. Off-ledger signers implement one envelope.

### Decimal prices

Prices are Decimal, as amounts are in [CIP-0056](../cip-0056/cip-0056.md), to
avoid the scale-conversion and rounding errors of scaled-integer representations.
`Decimal` has ten fractional digits. A feed that needs more quotes a scaled unit,
or falls back to a distributor schema carrying an integer coefficient and an exponent.

### A closed value type

The pull path reconstructs a payload on-ledger and checks a signature over its
hash, which requires exactly one encoding per value. A closed set of
constructors with type-tagged hashing gives each value a single hash and makes
the hash injective across types: without tags, `AVInt 1` and `AVText "1"`, or
an empty list and an empty map, would collide. The set is a subset of the
Token Standard's `AnyValue`; `Party`, `ContractId`, `Date` and `RelTime` are
left out because it is assumed that a payload signed off-ledger has no use
for contract ids and because the others are expressible with the remaining
scalars. The standard declares its own types instead of importing the Token
Standard's, so that it carries no dependency on token packages.

### Related work on Canton

Oracle designs that manage visibility and licensing through contract
hierarchies, such as Redstone's CAPS proposal, operate at the product layer.
Their consumer-facing contracts can implement `PublishedDataPoint`, so that
applications read them through the standard alongside other distributors.


## Reference implementation

The reference implementation is
[kaikodata/canton-data-standard](https://github.com/kaikodata/canton-data-standard),
release `v0.2.0`, built with Daml SDK 3.4.11 for Daml-LF 2.1. It contains:

- the four packages of the standard, with their built DARs committed under
  `dars/`;
- the reference producers and consumers listed in [Use cases](#use-cases);
- `tests`, covering the push interface without token or crypto dependencies;
- `tests-codecs`, holding the normative golden vectors of the structural hash;
- `tests-crypto`, covering the signature paths, including fixtures signed
  outside Daml with Python and with a Go signer;
- producer and consumer guides under `docs/`.

Development of the standard is funded in part by the Canton Foundation
Development Fund, via the
[Kaiko Data Standard](https://github.com/canton-foundation/canton-dev-fund/blob/main/proposals/2026-05-Kaiko-data-standard.md)
grant.


## Appendix: off-ledger hash implementation

Informative. Written from the [Signed envelope](#signed-envelope) section and
checked against the golden vectors of the reference implementation.

```python
import hashlib
from decimal import Decimal

def H(s):
    return hashlib.sha256(s.encode("utf-8")).hexdigest()

def node(hashes, tag=None):
    parts = ([tag] if tag is not None else []) + [str(len(hashes))] + hashes
    return H("|".join(parts))

def render_decimal(d):
    i, _, f = format(Decimal(d), "f").partition(".")
    return f"{i}.{f.rstrip('0') or '0'}"

def any_value(kind, x):
    if kind == "int":     return node([H(str(x))], "int")
    if kind == "decimal": return node([H(render_decimal(x))], "decimal")
    if kind == "text":    return node([H(x)], "text")
    if kind == "time":    return node([H(str(x))], "time")   # epoch milliseconds
    if kind == "bool":    return node([H("true" if x else "false")], "bool")
    if kind == "list":    return node([node([any_value(*e) for e in x])], "list")
    if kind == "map":     return node([map_hash(x)], "map")

def map_hash(m):
    return node([node([H(k), any_value(*m[k])]) for k in sorted(m)])

def root_hash(published_ms, expires_ms, schema_version, values):
    return node([H(str(published_ms)), H(str(expires_ms)), H(schema_version),
                 any_value("map", values)])

assert root_hash(1768480245000, 1768483845000, "quote-1", {
    "feedId": ("text", "BTC/USD"),
    "price": ("decimal", "65000.0"),
    "priceTime": ("time", 1768480245000),
}) == "156a45a913599d7597df9e065d9f5b3dd57b41d5910da81e148368c9061de19e"

# Signing: ECDSA secp256k1 with SHA-256 over bytes.fromhex(root), DER-encoded.
```

