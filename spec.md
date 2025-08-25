## Introduction

The Wallet Attached Storage (WAS) specification brings together the lessons
learned from numerous attempts to standardize permissioned cloud storage over
the years.

Initial use cases that are motivating this work:

* Sharing of W3C Verifiable Credentials from mobile credential wallets, as well
  as document sharing in general
* Enabling data portability and service provider interoperability for
  user-controlled social networking
* Providing data storage and authorization frameworks for Agentic AI
* Enabling the "bring your own storage" architecture pattern of web app development

This specification aims to provide:

* A set of design constraints, goals and requirements for WAS
* An HTTP API for reading and writing to permissioned cloud storage, to serve
  as a unifying interface for file and folder storage, object and bucket storage,
  as well as databases (including RDBMSs, document stores, graph stores, etc)
* An authorization and authentication framework for use with this storage

## Goals and Requirements

This storage specification is intended to support the following goals and
requirements.

### Local-first and Offline capable storage

Users and apps need to be able to use (provision, set up, and start reading and
writing to) storage spaces without being connected to the internet.

### Storage and sharing of public, permissioned, and private encrypted data

Although the local-first offline functionality is necessary, writing data to
stable internet-accessible URIs for the purposes of sharing them is one of the
primary use cases of this specification.

* A user needs to be able to write data (that is intended to be world-readable)
  to a cloud-accessible URI, and be able to send that URI to intended recipients
  via any out of band mechanism such as email, chat, and so on.

* User needs to be able to change or revoke permissions at any point after
  sharing. Note that changed permissions apply only to subsequent operations
  (this spec is not intended to solve the general problem of DRM).

* The sharing and permission system needs to be primarily based on authorization
  capabilities (zcaps), but it needs to also support storage-side access control
  list or similar functionality (even if only as a way for an authorized client
  to receive an appropriate zcap)

* The sharing mechanism needs to be flexible and granular. For example, a given
  data resource needs to be: world-readable, or readable by groups or categories,
  or by only those possessing the required authorization capabilities, or by 
  noone except the author or controller, etc

* Advanced sharing conditions are also desireable (such as "this share expires
  after X amount of time" or "this is a one-time share, and will expire after
  the first successful read request")

### Stored data is opaque to the storage provider

* The spec needs to support (though not require) end-to-end client side 
  encryption of the space. For plausible deniability, this might need to include
  all all data (even marked as public-readable) is encrypted at rest

### Replication to user-controlled local and cloud servers

* Replication reconciles the first two requirements (data reads and writes must
  be offline-capable, but the data must eventually be able to be shared on the
  web via traditional URIs)

* Replication also provides critical availability and disaster recovery 
  functionality

* Replication needs to be multi-primary (to reflect the multi-device and multi-
  client user environment)

* Multi-primary replication requires support for a versioning or conflict
  resolution mechanism

* Data, metadata, and permissions all need to be replicated

* Authorship and data provenance (the ability to tell which user or service
  created or edited a given set of data) must work in this permissioned multi-
  primary write environment

### Serve as a General Purpose application storage backend

Intended to serve as storage backend to client-side (Single Page Applications),
server side, desktop, and mobile apps and services.

### Data Portability

Data written to storage spaces using this specification needs to be portable:

* Authorized agents need to be able to export or backup all of the data written,
  including all corresponding metadata and permissions

* The sharing and storage system needs to be able to support web domain 
  independent identifiers. That is, a user must be able to share data at a given
  URI, then be able to migrate to a different storage service provider 
  (potentially operating on a different web domain than the previous one), and
  the shared permissions to that data must not break after service migration

* While portability (and the not breaking of URIs) is relatively easy to achieve
  via redirect mechanisms (such as HTTP 301 and 302 redirect codes), this 
  requires the previous service provider to be alive, available and cooperative.
  However, this is true only of public-readable URIs, and the moment permissions
  are involved, cross-domain redirects become very difficult.
  In addition, portability from "dead servers" is also required. That is, if a
  cloud based service provider disappears (or is otherwise unavailable), but a
  user still has a backup/export available, they should be able to set up another
  storage server (on another web domain or network address), and import/restore
  the data from backup, without shares and permissions breaking (agents that
  the data was previously shared with must still be able to find the data at
  the new storage server location, and their permissions must still work) 

### A Plurality of Data Formats and Protocols

* Spec needs to support the storage of any kind of data -- binary files and
  objects, structured documents such as JSON or CBOR, contents of relational
  database tables, graphs, and anything else, all using the same unified metadata,
  sharing and permission mechanisms.

* Storage-side schema enforcement is available but not required.

* Spec needs to be able to support multiple protocols and APIs, such as HTTP,
  JSON-RPC, DIDComm, local client APIs, and more.

### Permissioned Query and Search functionality

* Where appropriate (such as for unstructured text, structured documents, RDBMs
etc), storage needs to be queryable or searchable

* Any query/search mechanism needs to work well with the sharing/permission and
  replication requirements

### Upgradeable and legislation-compliant cryptography

All cryptography has a half-life.

* Any cryptographic operations (such as hashing, signatures, and encryption)
  used in this specification must be able to be obsoleted or upgraded, as
  techniques and algorithms break. To put it another way, the spec cannot
  "hardcode" any given algorithm (although it can recommend current best 
  practices)

* Implementations of this spec need to be usable in government and enterprise
  environments, which means all of the spec requirements must be able to be
  fulfilled using legislation-approved (for example, FIPS-compliant) in most
  jurisdictions

### Anti-Goals

#### Use cases do not include "zero trust" environments

This storage specification is intentionally positioned to not be used in
"zero trust" environments, which in practice means the usage of untrusted sync
and replication nodes while solely relying on encryption as the authorization
mechanism.

To put it a different way -- all encryption has an unpredictable half life, and
some use cases do not permit relying on encryption only for access control.
Instead, a _combination_ of encryption and authorization enforcement by
minimally trusted storage servers is required.

## Terminology

<dl class="termlist definitions" data-sort="ascending">
</dl>

## Authorization

The ability to do cross-domain, operator-independent, standardized cloud storage
operations requires an authorization system that is:

* Modular and layered (for future agility / upgradability)
* Not limited to traditional domain-based "usernames and passwords"
* A hybrid, using object capability principles at its baseline, but also able
  to provide ACL or RBAC-like functionality for user convenience. In other words,
  the system needs to support both "anyone with the link can..." and "these are
  specific people and groups allowed to..." styles of access control
* Compatible with cross-domain replication
* Compatible with end-to-end client side encryption (but also not rely on
  encryption as the sole authorization method)
* "Private by default".
  That is, by default, unless otherwise specified, only the controller of a
  space (or of a collection or resource) is authorized to perform any operation
  (read, write, delete, etc)

As the state of the art in cross-domain authorization advances, we expect there
to be multiple profiles and specs that could be used to perform WAS API
calls. However, to start with, this specification will focus on a single minimal
authorization profile.

## WAS Authorization Profile v0.1

Like many authorization specifications, the W.A.S. Authorization Profile tries
to address opposing tensions. On the one hand, to cover the full range of use
cases, it needs to be delegatable, revocable, secure, flexible, and thus
capability based. On the other hand, for ease of implementation and adoption,
and for maximum developer usability, the profile must make the most common
operations as simple and friction free as possible.

To that end, the profile offers the following layered mechanisms.

1. **Root Access**: For basic admin CRUD operations, use the space's `controller`
   DID directly to sign API calls with HTTP Signatures.
2. **Public Read**: For the common "public read" use case (the typical web
   publishing workflow, where a site or a file is shared for anyone to access
   via an HTTP GET), use the simple `publicRead: true` WAS Authorization syntax,
   see below.
3. **Advanced Delegatable Capabilities** ("anyone with the link..." style):
   Use zCaps [Authorization Capabilities v0.3](https://w3c-ccg.github.io/zcap-spec/)
4. **Policy Based Access Control** (including the familiar "share with this list
   of people or groups" style): Use the space's `linkset` property to point to
   a linkset that includes a URL to an access control policy document.

### Authorization Specification Dependencies at a Glance

The initial W.A.S. Authorization Profile uses the following specifications.

1. Identity (for controllers or clients/agents): [DID 1.0](https://www.w3.org/TR/did-1.0/)
2. Capability data model: [Authorization Capabilities for Linked Data v0.3](https://w3c-ccg.github.io/zcap-spec/)
3. Protocol for obtaining authorization: Out of scope (implementers are encouraged
   to use VC-API, OpenId4VP, OAuth2, or GNAP, as appropriate)
4. Proof of Possession / authorization invocation: HTTP Signatures.
   MUST - [RFC 9421 HTTP Message Signatures](https://www.rfc-editor.org/rfc/rfc9421.html),
   MAY - [HTTP Signatures (Cavage draft 12)](https://datatracker.ietf.org/doc/html/draft-cavage-http-signatures)
5. Access Control / Policy language data model: TBD (see <#16>)

### Space `controller` and the Root of Trust

Space `controller`s MUST be in the form of a [DID](https://www.w3.org/TR/did-1.0/).

For minimal compatibility, all WAS implementations MUST support the
[`did:key` DID Method](https://w3c-ccg.github.io/did-key-spec/), using the
Multikey encoding of `Ed25519` elliptic curve keys, as specified in the
[Multikey section of the CID spec](https://www.w3.org/TR/cid-1.0/#Multikey)
as the space `controller`.

When a space is created via an HTTP [POST](#http-api-post-spaces) or
[PUT](#http-api-put-space-space_id) operation, the controller for that space
is set, either implicitly or explicitly. 

Implicitly, if no `controller` is specified in the PUT or POST create space
request, the server MUST determine and set the `controller` from the corresponding
authorization headers of the request (for example, from the `Authorization` header
when using HTTP Signatures).

Explicitly, if a client specifies the `controller` as part of the payload
of the PUT or POST create space request, the server MUST check that the methods
(key IDs) used in the headers are authorized in the `capabilityInvocation`
section of the `controller`'s DID document.

See below in the [HTTP POST](#http-api-post-spaces) sections for examples of
`controller` determination and verification.

Conceptually, the space's controller serves as the root of trust and authorization
for any operations on the space or its collections or resources.
That is, any operation requiring an authorization MUST provide a chain of proof
all the way to the space controller, by one of the following:

1. Direct: Provide a root capability invoked directly by the controller, or
2. Delegated: Invoke a capability delegated to some other agent by the controller, or
3. Matching Policy: (if using any kind of access control policy mechanism) Match
   an authorization policy specified in the `linkset` property of the space. This
   resource is related to the space controller because initially, it can only be
   modified either by the controller or an authorized party delegated to by the
   controller.

### Performing Authorized API Calls

Unless otherwise explicitly allowed via access control policy (see below),
all W.A.S. API calls require authorization.

This can be done in one of two ways:

1. (for admin-like root access) Use the `controller` DID directly to sign
   HTTP API requests using the HTTP Signatures specification.
2. (for advanced delegatable use cases) Use HTTP Signatures in combination
   with [Authorization Capabilities v0.3](https://w3c-ccg.github.io/zcap-spec/),
   and include a capability invocation header in the API request.

### Specifying Access Policy With Space Link Sets

To set access control policy for a space, use the `linkset` property.

Example (fetching a space's link set):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset HTTP/1.1
Host: example.com
Accept: application/linkset+json
Authorization: Signature keyId="did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW#z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW" ...
```

Response:

```http
HTTP/1.1 200 OK
Content-type: application/linkset+json

{
  "linkset": [
    {
      "anchor": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
      "acl": [
        "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/acl",
        "media": "application/json"
      ]
    }
  ]
}
```

Example (fetching a specific policy document from the link set):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/acl HTTP/1.1
Host: example.com
Authorization: Signature keyId="did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW#z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW" ...
```

```http
HTTP/1.1 200 OK
Content-type: application/json

{ "type": "PublicCanRead" }
```

## Spaces

### Space Data Model

`Space` properties:

* `id` - Deterministically set by server if not provided.
* `type` - A sorted array of strings, MUST include the type `Space`.
* `name` (optional) - An arbitrary human-readable name for the space. Does not
  have to be unique.
* `controller` - A cryptographic identifier (a [DID](https://www.w3.org/TR/did-1.0/)) 
  of the entity that is authorized to perform operations on the space (or to
  delegate authorization to other entities)
* `linkset` (optional) - A URL (relative or absolute) to a resource which contains
  a set of links to auxiliary resources (such as to access control policy
  documents)

### Create Space operation

To create a Space:

* Perform an authenticated Create Space operation that includes a Proof of
  (cryptographic material) Possession via a mechanism such as HTTP Signatures.

* If an `id` is provided in the Create Space request, it must start with `urn:uuid`.
  If no `id` is provided, it will be generated by the storage server.

* A `controller` DID MUST be provided in the request body, and the request must
  demonstrate proof of cryptographic control of that DID.
    * If no `controller` is provided, the server MUST return an HTTP 400 error
      response
    * The signing DID (from the proof of possession signature) MUST match the
      Space's `controller`. This is how the root of trust is initially set up
      (see the [Space `controller` and the Root of
      Trust](#space-controller-and-the-root-of-trust) section for more details)

* (Optional, out of scope) A given storage provider MAY impose additional
  requirements in order to create a Space for a given controller, such as:
  - a Verifiable Credential representing a pre-arranged onboarding coupon
  - a proof of payment
  - a proof of membership in an organization

#### (HTTP API) POST `/spaces/`

To create a space via HTTP API:

```http
POST /spaces/ HTTP/1.1
Host: example.com
Accept: application/json
Content-type: application/json
Authorization: ...

{
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Example success response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Note that in the example above:

* the `id` was not specified in the body of the request, and so was generated by
  the server and returned in the response

#### Create Space Errors

Example error response (missing `controller` property):

```http
HTTP/1.1 400 Bad Request
Content-type: application/problem+json
Content-Language: en

{
  "type": "https://wallet.storage/spec#create-space-errors",
  "title": "Invalid Create Space request.",
  "errors": [
    {
      "detail": "'controller' property is required.",
      "pointer": "#/controller"
    }
  ]
}
```

Example error response (missing Proof of Possession signature):

```http
HTTP/1.1 401 Unauthorized
Content-type: application/problem+json
Content-Language: en

{
  "type": "https://wallet.storage/spec#create-space-errors",
  "title": "Invalid Create Space request.",
  "errors": [
    {
      "detail": "Valid proof of possession of the 'controller' DID must be provided."
    }
  ]
}
```

Example error response (invalid authorization - the signing DID in the `Authorization`
header does not match the DID specified in the `controller`):

```http
HTTP/1.1 403 Forbidden
Content-type: application/problem+json
Content-Language: en

{
  "type": "https://wallet.storage/spec#create-space-errors",
  "title": "Invalid Create Space request.",
  "errors": [
    {
      "detail": "The signing DID from the Authorization header must match the 'controller' DID in request body."
    }
  ]
}
```

Example error response (missing or insufficient onboarding material provided):

Example error response (invalid `id` provided):

Example error response (a space with the specified `id` already exists):

### Read Space operation

* Requires appropriate authorization (root zcap invoked by the space's controller,
  or a zcap granting permission to read a particular space)
* Returns the details for the specified space `id`
* Only includes the resources the requester is authorized to see

The format of the response is determined based on content negotiation.

#### (HTTP API) GET `/space/{space_id}`

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e HTTP/1.1
Host: example.com
Accept: application/json
Authorization: Signature keyId="did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW#z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW" ...
```

Example success response (the space is empty of contents):

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset",
  "totalItems": 0
}
```

Example success response (resources have been added to the space):

* Contents listed in a format inspired by [ActivityStreams 2](https://www.w3.org/TR/activitystreams-core/#collections)
  Collections
* No pagination used
* The requester is authorized to see two of these items

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset",
  "totalItems": 2,
  "items": [
    { "id": "https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/10f6672d-7a24-486b-9622-691007ded846" },
    { "id": "https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/b42ec4d8-6ead-486d-b70a-c25d2ce4dfc7" }
  ]
}
```

#### Read Space Errors

Example error response (missing or insufficient authorization):

A server MUST return the same error response in the case of missing or insufficient
authorization as it would for a missing/not found space.

```http
HTTP/1.1 404 Not Found
Content-type: application/problem+json
Content-Language: en
{
  "type": "https://wallet.storage/spec#read-space-errors",
  "title": "Space not found or insufficient authorization."
}
```

Example error response (space id not found):

```http
HTTP/1.1 404 Not Found
Content-type: application/problem+json
Content-Language: en
{
  "type": "https://wallet.storage/spec#read-space-errors",
  "title": "Space not found or insufficient authorization."
}
```

### List Spaces Operation

* Requires appropriate authorization (root zcap invoked by the controller of one
  or more spaces, or a zcap granting permission to read a one or more spaces)

* Lists only the spaces the requester is authorized to see

#### (HTTP API) GET `/spaces/`

Example request:

```http
GET /spaces/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response (requester has read access to at least one space):

```http
HTTP/1.1 200 OK
Content-type: application/json

...
```

Example success response (requester does NOT have access to any spaces):

```http
HTTP/1.1 200 OK
Content-type: application/json

...
```

Example error response (missing authorization):

### Update Space operation

* Requires appropriate authorization (root zcap invoked by the space's
  controller, or a zcap granting permission to write to a particular space)
* Allows to update the following fields:
  - `name`
  - `controller`
  - `linkset`

#### (HTTP API) PUT `/space/{space_id}`

Note that this is a _full_ update (partial updates via http `PATCH` verb might
be supported later). However, some fields may not be updated (like `id`) and so
may be omitted from the request payload.

Note that this operation is idempotent.

* A `controller` property is required in the PUT request body.

Example request (updating the `name` and `linkset` properties of a space):

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e HTTP/1.1
Host: example.com
Content-type: application/json
Accept: application/json
Authorization: ...

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "name": "Newly renamed space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset"
}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Example error response (missing or invalid authorization):

Example error response (invalid `id` provided, the space does not exist):

Example error response (client is attempting to change an immutable field like
the space `id`):

### Delete Space operation

* Requires appropriate authorization (root zcap invoked by the space's
  controller, or a zcap granting permission to write to a particular space)
* Deletes the space and all of the data (collections and resources) contained
  in it
* This operation is idempotent

#### (HTTP API) DELETE `/space/{space_id}`

Example request (no request body):

```http
DELETE /space/81246131-69a4-45ab-9bff-9c946b59cf2e HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

TODO: Decide whether subsequent GET requests to the same space should result in
a 410 Gone, or the usual 404.

Example error response (missing or invalid authorization):

Example error response (invalid `id` provided):

## Collections

A collection is a namespace for Resources, and a unit of configuration, within
a space.

In other storage systems, a collection is also known as: Directory, Folder,
RDBMS Table, Document Collection, Graph, WebAPI FileList, Bucket, Solid
Container, EDV Vault, DWN Collection, and so on.

The use of collections is optional -- when a Space is created, the default
collection is automatically created within that Space.

### Collection Data Model

Collection properties:

* `type` - A sorted array of strings, MUST include the type `Collection`.
* Links to any metadata objects controlled by the Wallet Attached Storage server
* Links to any metadata objects modifiable by the collection's controller

#### Collection JSON Representation (Activity Streams 2 profile)

Example empty collection, in AS2 format:

```js
{
  "type": ["Collection"],
  "totalItems": 0,
  "items": []
}
```

### Create or Update Collection operation

#### (HTTP API) PUT `/space/{space_id}/{collection_id}/`

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/ HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"]
}
```

```http
HTTP/1.1 201 Created
```

### Get Collection operation

* Returns the collection metadata and item contents

#### (HTTP API) GET `/space/{space_id}/{collection_id}/`

Example request (getting a collection by its id):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"],
  "totalItems": 0,
  "items": []
}
```

### Delete Collection operation

#### (HTTP API) DELETE `/space/{space_id}/{collection_id}/`

## Resources and Blobs

### Blob Data Model

A unit of data, in transit or at rest.
(As described in the [W3c FileAPI: Blob Interface](https://w3c.github.io/FileAPI/#blob-section)).

Blob properties:

* byte stream
* `size` - length of the byte stream in bytes
  - Note: Although size can be derived from bytes, it's useful to be able to
    have it up front (for the receiving system to decide to reject an upload
    based on quota / exceeding max size, etc).
* `type` (from the IANA mime type registry) - If not specified, defaults to
  `application/octet-stream`

### Resource Data Model

A resource is a named (addressable) object stored in a [=space=], with metadata.
The data model is derived from [W3C FileAPI: File 
Interface](https://w3c.github.io/FileAPI/#file-section), but with the addition
of a few crucial properties.

In similar storage systems, a resource is called "File", "Object", "Document",
"Row", "Graph", and so on.

Resource properties:

* `id`
* `name` - optional
* Links to any metadata objects controlled by the Wallet Attached Storage server
* Links to any metadata objects modifiable by the resource's controller

### Read Resource Operation

* TODO: Add language on content negotiation

#### (HTTP API) GET `/spaces/{space_id}/{collection/*}{resource_name}`

* Requires appropriate authorization
  - For example, when using [zCAPs](#zcap) for authorization, the request
    must either: be signed by the resource's or the space's [=controller=],
    or invoke a delegated capability that allows the [`GET` action](#get-action)

Example request to retrieve a resource:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Accept: application/json
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{"message":"hi"}
```

* TODO: Add example 404 error response where a missing or invalid resource is
  specified, or if the request carries insufficient or missing authorization

### Update Resource Operation

#### (HTTP API) PUT `/spaces/{space_id}/{collection/*}{resource_name}`

* Requires appropriate authorization
  - For example, when using [zCAPs](#zcap) for authorization, the request
    must either: be signed by the resource's or the space's [=controller=],
    or invoke a delegated capability that allows the [`PUT` action](#put-action)
* This operation is idempotent
* Returns a `204` success response

Example request to update a resource via PUT to the `messages` collection:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Content-Type: application/json

{"message":"hi"}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

* TODO: Add example 404 error response where a missing or invalid space or
  collection is specified, or if the request carries insufficient or missing
  authorization
* TODO: Add example "over storage quota" error response

### Delete Resource Operation

#### (HTTP API) DELETE `/spaces/{space_id}/{collection/*}{resource_name}`

* Requires appropriate authorization
  - For example, when using [zCAPs](#zcap) for authorization, the request
    must either: be signed by the resource's or the space's [=controller=],
    or invoke a delegated capability that allows the
    [`DELETE` action](#delete-action)

* This operation is idempotent

* (Assuming the request carries appropriate authorization) Sending a DELETE
  request to a resource that does not exist (or has already been deleted)
  results in a 204 success response

Example request to delete a resource via DELETE:

```http
DELETE /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
```

Example success response:

```http
HTTP/1.1 204 No Content
```

* TODO: Add example 404 error response if the request carries insufficient or
  missing authorization

## Security and Privacy Considerations

## Internationalization Considerations

### Language and Base Direction

<section class="appendix informative">

## IANA Considerations

This section will be submitted to the Internet Engineering Steering Group (IESG)
for review, approval, and registration with IANA.
</section>

