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
  providers, benchmark administrators, fund administrators publishing NAVs,
  or market makers.
- **consumers**: Daml applications whose workflows depend on that data. For
  example lending protocols, DvP and settlement apps, collateral management, or
  tokenized fund subscription and redemption workflows.

A distributor publishes by implementing a standard interface on a template it
signs. A consumer reads through that interface and never names the
distributor's package. Integration is done once against the standard;
selecting a distributor is a choice of party.

The standard supports two delivery models:

- **Push**: the distributor creates one contract per publication, implementing
  `PublishedDataPoint`, on a template it signs.
- **Pull**: the distributor signs payloads off-ledger and hands them to
  consumers through its own channel. On-ledger, it publishes one long-lived
  contract implementing `DistributorKey`. A consumer authenticates any number
  of payloads against that key by calling the verification library from its
  own choice.

Both models carry the same payload type and the same schema identifiers, so
everything a consumer does after the read is shared between them.

The standard fixes what a consumer reads and how the consumer authenticates
it. Request flows, the transport of disclosed contracts and signed payloads,
pricing, entitlements and receipts belong to each distributor's product. A
standard for delivery can follow in a separate CIP.

### Packages

| Package | Content | Upgrade model |
|---|---|---|
| `canton-data-standard-utils-v1` | Shared types: `AnyValue`, `Values`, `Metadata`, `SignedDataPoint`, `VerifiedDataPoint`, and the `insertField` / `lookupField` helpers | No templates or interfaces; evolves by smart-contract upgrade (SCU) |
| `canton-data-standard-datapoint-v1` | The `PublishedDataPoint` interface | Interface package: never upgraded; a breaking change is a new `-v2` package |
| `canton-data-standard-distributor-key-v1` | The `DistributorKey` interface | Interface package, as above |
| `canton-data-standard-codecs` | Structural hash, signed envelope, signature verification, `Quote` codec | Utility package built with `--force-utility-package`: any two versions are SCU-compatible |

Interface packages contain views and one read choice each, and no other code.
A package that defines an interface cannot be upgraded, so code placed there
could never be fixed. Hashing, verification and codecs therefore live in the
codecs package, and `utils-v1` holds only types and small helpers. Every
package depends only on `daml-prim`, `daml-stdlib` and other packages of the
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
syntax of [CIP-0056](../cip-0056/cip-0056.md#metadata-key-syntax) and SHOULD
carry a DNS prefix identifying the organization that defines them (for example
`exampleoracle.com/methodology`). Entries SHOULD be small. Consumers MUST
ignore keys they do not recognize. A distributor adds an annotation as a
metadata entry rather than by changing a view.

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
  the pair (`distributor`, `schemaVersion`) identifies the schema. A
  distributor increments the patch version for clarifications, the minor
  version for added fields, and the major version for a renamed, removed or
  retyped field. Distributors SHOULD publish documentation of every schema they
  use.

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
implementation (`tests-codecs`) are normative. Two of them:

| Payload | Root hash |
|---|---|
| `publishedAt` = 1768480245000, `expiresAt` = 1768483845000, `schemaVersion` = `1.0.0`, `values` = {`feedId`: `AVText "EUR/USD"`, `rate`: `AVDecimal 1.0875`} | `1ac5fa4ba0227c3e348464699c40b5f313ffbae4c12943e544d2e44d176b575c` |
| `publishedAt` = 1768480245000, `expiresAt` = 1768483845000, `schemaVersion` = `quote-1`, `values` = {`feedId`: `AVText "BTC/USD"`, `price`: `AVDecimal 65000.0`, `priceTime`: `AVTime 1768480245000`} | `156a45a913599d7597df9e065d9f5b3dd57b41d5910da81e148368c9061de19e` |

An informative implementation of the hash, written from this section alone and
checked against the golden vectors, is given in the
[appendix](#appendix-off-ledger-hash-implementation).

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
the key, never from the payload, whose leading fields mirror
`PublishedDataPointView`, and whose last three fields are `canonicalHash` (the
root hash), `signature` and `publicKey`. Together with `expiresAt`, which the
result does not carry and which a consumer keeps if it needs to, they are the
evidence needed to reproduce the check off-ledger. `canonicalHash` is a stable
join key for any record that refers to the verification. There is no
`metadata` field: the envelope signs none, and annotations delivered alongside
a signed payload are not authenticated.

`verifyDataPointWith` performs the last three checks against a distributor and
key the caller supplies, for callers that obtain the key by other means than a
`DistributorKey` contract. The failure texts are part of the public surface of
the codecs package.

A consumer runs verification inside its own choice, and every validating
participant re-executes it as it would an interface choice. A defect in it is
fixed by a codecs release, which each consumer adopts by rebuilding and
upgrading its own package; the interfaces and the distributors'
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
  verifying transaction, and verify before decoding, so that no decoding runs
  on unauthenticated values. `expiresAt` is the standard's only replay bound;
  a consumer SHOULD apply its own, tighter freshness policy on top of it.
- SHOULD vet only implementation packages of distributors they trust, and
  SHOULD NOT accept a publication or key contract id from a counterparty
  without checking it (see [Security considerations](#security-considerations)).

### Visibility

A consumer reads only contracts it can see, and the standard supports two ways
of making a publication visible:

- **Explicit disclosure.** The distributor hands the consumer the
  `template_id`, `contract_id` and `created_event_blob` off-ledger, and the
  consumer attaches them to its command submission. Storage does not grow with
  the audience, and disclosure is tamper-evident because a contract id is a
  hash of the contract's contents. Disclosed contracts do not enter the
  consumer's active contract set and so are not indexed by the consumer's
  Participant Query Store (PQS).
- **Observers.** Naming consumers as observers makes them stakeholders: the
  publication reaches their participant, they can query it and `fetch` it
  directly. Storage and streaming then grow with the audience, which suits
  small, known audiences or consumers that discover publications by query.

Where PQS applies, one query by interface view covers every implementation of
`PublishedDataPoint`. Interface views are served on the transaction stream and
the active contract set, not on the transaction tree stream. On Canton 3.4,
PQS 3.4.3 or later fixes an interface view projection issue. The pull path
writes nothing per payload, and the one `DistributorKey` a consumer needs is
usable by disclosure alone.

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


### Security considerations

- **Distributor binding.** A view is computed by the implementing template, so
  the interfaces cannot by themselves prevent a template from naming a
  `distributor` that is not its signatory. The standard requires
  implementations to bind `distributor` to a signatory, and Canton executes
  only packages that the informees' participants have vetted, so a consumer
  that vets an implementation package trusts it to honour that binding. The
  risk is highest where a counterparty supplies the contract id, since it can
  create a publication or key of its own on any template the consumer has
  vetted. A consumer limits it by taking contract ids from the distributor
  rather than the counterparty, or by checking the template id of a disclosed
  contract against an allowlist before submission. See
  [Open issues](#open-issues).
- **Injectivity.** The security of the pull path rests on the structural hash
  being injective: a collision would let a signature be reused for a payload
  the distributor never signed. Type tags on `AnyValue` nodes and count
  prefixes on every node exist for this reason.
- **Replay.** A captured payload and signature verify until `expiresAt`, in
  any transaction. Consumers apply their own freshness bounds.
- **Time precision.** The hash commits to milliseconds, which is why signed
  payloads carry whole milliseconds. Verification does not yet reject finer
  times, which a relayer could alter below the millisecond without
  invalidating the signature; rejecting them is a codecs fix.
- **Key custody and rotation.** A compromised key signs valid payloads until
  its `DistributorKey` contract is archived. How consumers learn of a rotation
  is the distributor's concern.
- **Read visibility.** Each read informs the distributor, as described in
  [Visibility](#visibility). Consumers for whom the fact of a read is
  sensitive take this into account when choosing a distributor and a path.
- **Cryptographic builtin.** Signature verification uses Daml's
  `DA.Crypto.Text.secp256k1` builtin, which is marked alpha in Daml SDK 3.4.
  Only the codecs package calls it, so a change to the builtin is handled in a
  codecs release; packages that call the verify functions need no compiler
  flag.


## Motivation

Every financial application on Canton consumes data: a lending protocol needs
collateral prices, a settlement app a reference rate, a tokenized fund its NAV.
Today each application integrates each data source bespoke, against that
source's templates. Adding a second source or switching sources means new Daml
code and a new package dependency, and data providers entering the network face
one bilateral integration per application. Several data providers already
operate Super Validators.

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
| Paid data and receipts | A product template holds the key, exposes the `DistributorKey` view and settles a fee in its own choice; a receipt pins the `canonicalHash` | Product-level; not part of the standard |

Applications this enables include DvP settlement at a reference price chosen
from a trusted distributor, collateral valuation and liquidation thresholds,
NAV-based fund subscriptions and redemptions, index and basket payloads
carrying their constituents, and N-of-M agreement checks across distributors.


## Rationale

### Relation to oracle interfaces on other chains

Price feed interfaces on EVM chains, such as Chainlink's `AggregatorV3Interface`
(`latestRoundData()` returning an integer answer and its update time) and
[ERC-7726](https://eips.ethereum.org/EIPS/eip-7726) (`getQuote(baseAmount,
base, quote)`), standardize a read call on a feed contract. The push path is
the Canton counterpart: `publishedAt` corresponds to the update time, the pair
(`distributor`, `feedId`) to the feed contract, and `price` to the answer,
carried as a `Decimal` instead of an integer with decimals. Unlike ERC-7726,
the view carries the publication time, and staleness policy is left to the
consumer. Signed-report oracles, where a consumer verifies a provider's
signature on-chain, correspond to the pull path. Two differences follow from
Canton's model: the consumer reads a contract by disclosure or as a
stakeholder rather than from global state, and the payload is a typed map
rather than a single number.

### One payload interface instead of a typed quote interface

The Development Fund proposal for this standard and earlier drafts defined a
separate `PublishedQuote` interface. A quote is a payload of three fields, so
it now travels the same interface and the same signed envelope as every other
payload, under `quote-1`. A typed record frozen into an interface package
could never be corrected, while the `Quote` codec in the utility package can.
Off-ledger signers implement one envelope. `DistributorKey` was added in its
place to cover the pull delivery model, which the proposal named but the push
interface alone does not serve.

### Decimal prices

Prices are `Decimal`, as amounts are in [CIP-0056](../cip-0056/cip-0056.md),
to avoid conversion errors between integer mantissas and decimals. `Decimal`
has ten fractional digits. A feed that needs more precision quotes a scaled
unit or uses a distributor schema with an integer mantissa and an exponent.

### A closed value type

The pull path reconstructs a payload on-ledger and checks a signature over its
hash, which requires exactly one encoding per value. A closed set of
constructors with type-tagged hashing gives each value a single hash and makes
the hash injective across types: without tags, `AVInt 1` and `AVText "1"`, or
an empty list and an empty map, would collide. The set is a subset of the
Token Standard's `AnyValue`; `Party`, `ContractId`, `Date` and `RelTime` are
left out because a payload signed off-ledger has no use for contract ids and
the others are expressible with the remaining scalars. The standard declares
its own types instead of importing the Token Standard's, so that it carries no
dependency on token packages.

### Verification as library code

Earlier drafts defined verifier interfaces, including paid and audited
variants, whose implementations ran the signature check and created audit
records. Moving verification into a library called from the consumer's choice
removes per-payload contract writes and contention on a shared verifier
contract, keeps the verification code fixable, and leaves fee settlement and
receipts to products, which differ between distributors. The evidence fields
of `VerifiedDataPoint` are what a receipt needs.

### Structural hash instead of a canonical byte encoding

Reusing the combinators of `Splice.Amulet.CryptoHash` gives a scheme that is
already defined in Splice, that Daml computes with the stable `DA.Text.sha256`,
and that other languages implement with SHA-256 and string joins. Field
boundaries come from nesting and count prefixes, so there is no delimiter
escaping. The golden vectors were produced by a separate Python implementation
written from the scheme definition, signatures from a Go signer are verified
against them in the reference test suite, and the implementation in the
[appendix](#appendix-off-ledger-hash-implementation), written from this CIP
alone, reproduces them.

### Scope left to products

The standard defines no fees, entitlements, delivery channels or reward hooks.
Distributors monetize through their own templates, for example by settling a
fee in the choice that holds their key or by creating featured app activity
markers ([CIP-0047](../cip-0047/cip-0047.md)) in their own choices. Keeping
these out of the interfaces lets distributors with different commercial models
implement the same read surface.

### Related work on Canton

Oracle designs that manage visibility and licensing through contract
hierarchies, such as the CAPS proposal, operate at the product layer. Their
consumer-facing contracts can implement `PublishedDataPoint`, so that
applications read them through the standard alongside other distributors.

### Review and consensus

The design and the reference repository were reviewed with Digital Asset.
Following that review, the surface was reduced from ten interface packages to
the two in this CIP, verification moved into library code, and paid
verification and audit records moved to the product level. Design sessions
were held with Cumberland (DRW). Discussions with further oracle providers are
ongoing, and their feedback is recorded here before the CIP is proposed for a
vote.


## Backwards compatibility

The standard is new and changes no existing Splice API.

Implementations built against the pre-release packages of the reference
repository rebuild against the packages published in Splice once their final
names are fixed, because a package name change produces new package ids. Their
Daml source changes only in the imported module names.

Future versions follow the [Evolution](#evolution) rules: a `-v2` interface
package coexists with `-v1`, so implementations and consumers migrate on their
own schedule.


## Reference implementation

The reference implementation is
[kaikodata/canton-data-standard](https://github.com/kaikodata/canton-data-standard),
release `v0.2.0`, built with Daml SDK 3.4.11 for Daml-LF 2.1. It contains:

- the four packages of the standard, with their built DARs committed under
  `dars/` and checked in CI against a rebuild from source (package id
  comparison);
- the reference producers and consumers listed in [Use cases](#use-cases);
- `tests`, covering the push interface without token or crypto dependencies, so
  that the suite also runs against a live Canton ledger;
- `tests-codecs`, holding the normative golden vectors of the structural hash;
- `tests-crypto`, covering the signature paths, including fixtures signed
  outside Daml with Python and with a Go signer;
- producer and consumer guides under `docs/`.

Remaining work before this CIP is given status Final: the merge of the packages
into Splice under their final names, and the publication of the developer
documentation.

Development of the standard is funded in part by the Canton Foundation
Development Fund, via the
[Kaiko Data Standard](https://github.com/canton-foundation/canton-dev-fund/blob/main/proposals/2026-05-Kaiko-data-standard.md)
grant.


## Open issues

These points are open for discussion on cip-discuss and are resolved before the
CIP is proposed for a vote.

- **Distributor binding in the read choices.** `PublishedDataPoint_Fetch` and
  `DistributorKey_Fetch` could assert that the view's `distributor` is among the
  contract's signatories. This one-line check would make the binding hold
  independently of package vetting, at the cost of code in packages that
  cannot be upgraded.
- **Envelope binding.** The envelope commits to the payload and its validity
  window, but not to the distributor party, the codec identifier or a domain
  separation tag. A public key published by two parties verifies under both.
  Adding them to a new envelope would be a new codec identifier and leaves the
  interfaces unchanged.
- **Evidence completeness.** `VerifiedDataPoint` omits `expiresAt`, which is
  needed to recompute `canonicalHash`. Adding it as an optional field is an
  upgrade of `utils-v1`.
- **Quorum-signed payloads.** Oracle networks that sign reports with a set of
  nodes, of which a threshold must agree, need verification against several
  keys of one distributor. This can be added as library code over several
  `DistributorKey` contracts, but where the threshold is defined (by the
  distributor on-ledger, or by the consumer) is open.
- **Native report codecs.** `payloadCodec` allows codecs that verify a
  provider's existing signed report format and return a `VerifiedDataPoint`,
  so that a provider does not re-sign its data in the standard envelope.
  Whether such codecs belong in the standard's codecs package or in providers'
  own libraries is open.
- **Metadata on `DistributorKeyView`.** `PublishedDataPointView` carries
  `metadata`; `DistributorKeyView` does not. Adding it before the interfaces are
  published in Splice keeps additive evolution available for keys.
- **Value types and the Token Standard.** Whether `utils-v1` keeps its own
  `AnyValue` and `Metadata` or reuses those of `splice-api-token-metadata-v1`.


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


## Changelog

- **2026-10-05**: Initial draft.
