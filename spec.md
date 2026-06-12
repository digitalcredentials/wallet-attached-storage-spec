## Introduction

The Wallet Attached Storage (WAS) specification brings together the lessons
learned from many attempts to standardize permissioned cloud storage over
the years.

This specification aims to provide:

* A set of **design constraints, goals, and requirements** for WAS, see **Appendix
  [[[#goals-and-requirements]]]**.
* A tiered composable **data model** for storage primitives
* An **HTTP API** binding for storage operations (other bindings, such as JSON-RPC
  or CBOR-based RPCs, are left for future work).
* An authorization profile for use with this storage, see **Appendix
  [[[#was-authorization-profile-v0-1]]]**.

### Use Cases

Initial use cases that are motivating this work:

* Sharing of W3C Verifiable Credentials from mobile credential wallets, as well
  as document sharing in general
* Serving as standardized storage (and a point of sync and interop) for
  multiple Verifiable Credential wallets.
* Enabling data portability and service provider interoperability for
  user-controlled social networking
* Enabling the "bring your own storage" architecture pattern of web app development
* Providing data storage and authorization frameworks for Agentic AI

### Scope and Conformance Profiles

This specification represents a layered and modular approach to storage, combining
core features and optional extension points.

**Permissioned Key/Value CRUD**: At its core, this spec is an example of how to
implement a permissioned key/value CRUD API using simple HTTP verbs and delegatable
capability-based authorization. Implementers can use this layer (just sections
[[[#resources-and-blobs]]] and [[[#was-authorization-profile-v0-1]]]), without
needing collection or space management, for reads and writes of any resource
(text, structured document, or binary blob).

**Listing items**: Some use cases, like the famous "implement a blog with comments
in 15 minutes" framework posts, need to list items in a container or collection
of some kind. This spec follows a general pattern of "here are the listing endpoints,
send a permissioned GET to one, and receive back some items":

* GET `/space/{space_id}/{collection_id}/` endpoint to list items ([[[#list-collection-operation]]])
* GET `/space/{space_id}/collections/` endpoint to list collections in a space
  ([[[#list-all-collections-operation]]])
* GET `/spaces/` endpoint to list spaces in a server ([[[#list-spaces-operation]]])

And just as with the previous layer, some collections contain structured data,
and some contain just binary blobs (files or objects).

**Collections management**: On the next layer, clients can create and manage
arbitrary collections for a given space. If you've ever used a GUI front end
for a database or file system, you'll be familiar with these operations.
(See section [[[#collections]]].)

**Spaces management**: Some implementations, like cloud storage providers,
may support multi-tenancy at the space level, adding the concept of [[[#spaces-repositories]]],
allowing a client to create and manage multiple spaces.

**Extensions**: Linksets (from [[RFC9264]]) serve as a feature detection and
extension mechanism. See Appendix [[[#linksets]]] for more details..
Linksets support optional features such as:

* The `policy` auxiliary resource provides a way to specify arbitrary access policies
* User-writable metadata lets clients attach arbitrary data to resources (for
  example, user-defined "tags" for binary files)
* A space's Export endpoint allows a client to download a full backup of the
  data contained in a space
* The concept of [[[#backends]]] as storage engines for collections
* Query and search: some Backends support querying or search capability
* Quota management: some Backends support quota limit enforcement
* Client-side Encryption, using the [Encrypted Data Vaults](https://identity.foundation/edv-spec/)
  spec as a backend
* Replication and Sync: when supported, a client can set up multiple replicas
  for a given collection, depending on backend support
* Versioning: similarly, depending on backend support, some resources can have
  versioning (and in case of replication, conflict resolution) capabilities

### API Summary

API summary at a glance.

**Unless otherwise specified** by the [=controller=], all **operations
require authorization**. This can be overridden by the controller via the `policy`
policy endpoints. Hosting "public-read" resources, such as HTML files for websites,
or media files you can link to via `<img src="">`, is a common and valid use
case.

**(Core) Resource CRUD (Create, Read, Update, Delete):**

* `POST /space/{space_id}/{collection_id}/` - Create a new resource (add to a Collection)
* `GET /space/{space_id}/{collection_id}/{resource_id}` - Read a resource
* `PUT /space/{space_id}/{collection_id}/{resource_id}` - Update a resource (create or replace)
* `DELETE /space/{space_id}/{collection_id}/{resource_id}` - Delete a resource
* Use a `HEAD` instead of a `GET` to check a resource's headers/metadata without
  fetching the whole resource. However, do not rely on this for atomic inserts
  (that is, to check if a resource exists before attempting to create it).
  Instead, use the backend-specific Transaction mechanisms when appropriate.

**List resources in a Collection (Optional):**

If not implemented on a server, implies that only individual Key/Value operations
are supported.

* `GET /space/{space_id}/{collection_id}/` - List all resources in a Collection

**Manage Collections in a Space (Optional):**

If not implemented on a server, implies that collections are pre-configured or
implicit, controlled by the server.

* `POST /space/{space_id}/collections/` - Create a new Collection
* `GET /space/{space_id}/collections/` - List all Collections in a Space
* `GET /space/{space_id}/{collection_id}` - Get the Collection Description object (properties)
* `PUT /space/{space_id}/{collection_id}` - Update a Collection's properties
* `DELETE /space/{space_id}/{collection_id}` - Delete a Collection and its contents

**Spaces Repository Endpoints - Manage Spaces on a Server (Optional):**

If not implemented on a server, implies that any existing Spaces are
pre-configured and controlled by the server.

* `POST /spaces/` - Create a new Space
* `GET /spaces/` - List all Spaces (that the requester is authorized to see)

**Space Endpoints - Manage an individual Space (Optional):**

* `GET /space/{space_id}` - Get the Space Description object (properties)
* `PUT /space/{space_id}` - Update a Space's properties
* `DELETE /space/{space_id}` - Delete a Space and its contents
* `POST /space/{space_id}/export` - Export (download) a Space's contents (all
  collections and resources)

**Advanced Resource Endpoints (Optional):**

* `GET /space/{space_id}/{collection_id}/{resource_id}/meta` - Get the detailed Metadata object for a resource
* `PUT /space/{space_id}/{collection_id}/{resource_id}/meta` - Update user-writable Metadata properties for a resource

**Policy Related Endpoints (Optional):**

Policy overrides are hierarchical and inherited. A policy set for the entire
Space applies to all its Collections and Resources (unless overridden by a
more specific policy, either at the Collection or Resource level).

* `GET|PUT|DELETE /space/{space_id}/policy` - CRUD on the policy object for the Space
* `GET|PUT|DELETE /space/{space_id}/{collection_id}/policy` - CRUD on the policy object for the Collection
* `GET|PUT|DELETE /space/{space_id}/{collection_id}/{resource_id}/policy` - CRUD on the policy object for the Resource

**Linkset / Discovery Endpoints (Optional):**

Required if Space endpoints or Collection endpoints are supported.

* `GET /space/{space_id}/linkset` - Get the Linkset object for the Space
* `GET /space/{space_id}/{collection_id}/linkset` - Get the Linkset object for the Collection

**Query Endpoints (Optional):**

* `POST /space/{space_id}/query` - Reserved for cross-collection queries (backend-specific)
* `POST /space/{space_id}/{collection_id}/query` - Reserved for queries within a Collection (backend-specific)

**Backend and Quota Management Endpoints (Optional)** (see Appendix
[[[#backends]]]):

* `GET /space/{space_id}/backends` - Get the list of available backends for a Space
* `GET /space/{space_id}/quotas` - Get the Quota report object, grouped by available Backend
* `GET /space/{space_id}/{collection_id}/backend` - Get the detailed backend object for a Collection
  (the backend summary will also be displayed in the Collection description object)
* `GET /space/{space_id}/{collection_id}/quota` - Get the Quota report object for
  the specific Collection (not all Backends will support per-collection quotas however)

### Error Handling

This specification uses [[RFC9457]] Problem Details for HTTP APIs for error responses.

* Error responses SHOULD be returned using the `application/problem+json` content type.
* `type` and `title` properties are REQUIRED.
* `errors` array with `{ detail, pointer }` objects is encouraged where appropriate.

The `type` property is a URI identifying the _kind_ of problem (not the
operation), and the same `type` is reused across operations. See Appendix
[[[#error-type-registry]]] for the catalog of `type` URIs this specification
defines, along with their typical status codes and a canonical example response
for each kind.

When returning errors, keep in mind the principle of **maximum privacy**:
always "not found" instead of "not authorized". The _existence_ of a resource,
collection, or space, is by itself an item of sensitive information.
If a client makes an API call, and they have insufficient authorization to
perform that action, a "not found" error (such as HTTP 404) MUST be returned,
just as if that resource (or space or collection) did not exist.
To put it another way, an unauthorized client (meaning, either not carrying
any authorization in the request itself, or possessing insufficient permissions)
MUST NOT be able to discover the existence of a resource based on the error
response.

This privacy rule governs failures that would otherwise reveal whether a target
*exists*. It does not require masking failures that describe the *request
itself*:

* **Request and credential validation** failures: a malformed or missing
  request body, a missing `Content-Type`, or a capability invocation or
  signature that is present but malformed or fails to verify -- MAY be reported
  with precise status codes and `type`s (for example [=invalid-request-body=],
  [=missing-content-type=], [=invalid-authorization-header=], or
  [=controller-mismatch=]). These responses describe the request and do not
  disclose whether any particular target exists.
* **Authorization** failures against an existing target: a well-formed request
  that simply lacks sufficient privilege, including one that carries no
  authorization at all, MUST be reported as [=not-found=] (HTTP 404),
  indistinguishable from the target being absent.

"List Spaces" operations are an exception to 404 masking: rather than returning
an error, they return `200 OK` with only the subset of items the caller is
authorized to see (an empty `items` array if none) -- see [[[#list-spaces-operation]]].

### Caching

TODO: Add caching semantics section.

* `ETag`, `If-None-Match`, `Last-Modified`, and `Cache-Control` are encouraged
* Set "no cache" headers for non-idempotent operations

### Pagination

TODO: Add pagination section for 'list spaces', 'list collections', and 'list resources'
operations.

## Terminology

<dl class="termlist definitions" data-sort="ascending">
  <dt><dfn data-lt="action|actions|allowedAction">action (allowedAction)</dfn></dt>
  <dd>The kind of operation a request performs on a target, named by a capability
    so it can be authorized. WAS uses the uppercase HTTP method names
    (<a>GET</a>, <a>POST</a>, <a>PUT</a>, <a>DELETE</a>) as its action
    vocabulary. See section
    [[[#authorization-actions-and-the-root-capability]]].</dd>

  <dt><dfn data-lt="collections">collection</dfn></dt>
  <dd>A namespace and configuration container for resources. Conceptually maps
    to folders (for file system like storage), buckets (for object storage), or
    database tables (for RDBMSs). See section [[[#collections]]].</dd>

  <dt><dfn data-lt="controllers">controller</dfn></dt>
  <dd>An entity that has the capability to make changes to a given object.</dd>

  <dt><dfn data-lt="did|dids">decentralized identifier (DID)</dfn></dt>
  <dd>See [[DID-CORE]].</dd>

  <dt><dfn data-lt="server|servers|instance|instances">instance, server</dfn></dt>
  <dd>A deployed instance of an application or service that implements this
    specification's API.</dd>

  <dt><dfn data-lt="root capabilities|root zcap">root capability</dfn></dt>
  <dd>The implied capability for a [=target=] whose [=controller=] is the Space's
    controller; it is the root of trust from which all other capabilities for
    that target are delegated. See section [[[#root-capability]]].</dd>

  <dt><dfn data-lt="target|invocationTarget|targets">target (invocationTarget)</dfn></dt>
  <dd>The resource a request acts on -- the full request URL (scheme, host, port,
    and path) -- and the scope a capability authorizes. A capability's
    `invocationTarget` MUST match the request target for the invocation to be
    valid. See section [[[#authorization-actions-and-the-root-capability]]].</dd>

  <dt><dfn data-lt="zcap|zCaps|capability|authorization capability">zCap (Authorization Capability)</dfn></dt>
  <dd>See [zCap Developer Guide](https://interop-alliance.github.io/zcap-developer-guide/) for more
    details.</dd>
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

See **Appendix [[[#was-authorization-profile-v0-1]]]** for a description of the
default profile based on Authorization Capabilities (zCaps).

## Spaces Repositories

A Spaces Repository is a set of API endpoints that supports the creation and
management of multiple spaces on a given [=server=].
This `/spaces/` set of API endpoints is **optional**. If a server does not support
this feature (for example, if it is a single-tenant server with an existing
hardcoded Space), then it can implement only the `/space/{space_id}/` endpoints
and get most of the functionality of this specification.

<div class="ednote">
The Spaces Repository endpoints are a candidate for extraction into a
standalone companion specification (multi-tenant space provisioning) in a
future version of this document.
</div>

### Create Space operation

To create a Space:

* Perform an authorized Create Space operation that includes a Proof of
  (cryptographic material) Possession via a mechanism such as HTTP Signatures.

* If no `id` is provided, it will be generated by the storage server.

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

To create a space via HTTP API using the Spaces Repository POST API:

```http
POST /spaces/ HTTP/1.1
Host: example.com
Accept: application/json
Content-type: application/json
Authorization: ...

{
  "type": ["Space"],
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

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=invalid-request-body=] (400) -- the required `controller` property is
  missing from the request body.
* [=missing-authorization=] (401) -- the request lacks a valid proof of
  possession of the `controller` DID.
* [=controller-mismatch=] (400) -- the signing DID in the `Authorization` header
  does not match the `controller` DID in the request body.
* [=invalid-id=] (400) -- the supplied Space `id` is not URL-safe (see
  [[[#identifiers]]]).
* [=id-conflict=] (409) -- a Space with the supplied `id` already exists.

Onboarding requirements, if any, are provider-specific and out of scope for this
specification (see the optional onboarding material above). Their error `type`,
`title`, and status code are likewise provider-defined.

A `POST /spaces/` that supplies an `id` already in use returns the
[=id-conflict=] error. To create or replace a Space at a client-chosen
`id` without conflict, use the idempotent
[[[#update-or-create-by-id-space-operation]]] instead.

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

{
  "url": "/spaces/",
  "totalItems": 1,
  "items": [
    {
      "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e"
    }
  ]
}
```

Example success response (requester does NOT have access to any spaces):

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "url": "/spaces/",
  "totalItems": 0,
  "items": []
}
```

A request that carries no authorization (or authorization for no spaces) is not
an error: like any list operation it returns the `200 OK` empty-list response
shown above, revealing nothing about which spaces exist (see
[[[#error-handling]]]).

## Spaces

A space is a namespace for collections and a unit of general configuration,
a volume of storage that contains one or more collections.
Conceptually, is maps to a disk partition (for file systems), or a database
(for relational databases).

### Space Data Model

`Space` properties:

* `id` - A unique identifier for a space at a given [=server=]. Created by the
  server if not provided. Note: the `{space_id}` template parameter used in URL
  templates in this spec MUST match that space's `id` property. See Appendix
  [[[#identifiers]]] for additional constraints.
* `type` - An array of strings, MUST include the type `Space`. The array SHOULD
  be lexically sorted so that the object has a canonical, stable serialization
  (useful for hashing or signing).
* `name` (optional) - An arbitrary human-readable name for the space. Does not
  have to be unique.
* `controller` - A cryptographic identifier (a [=did=])
  of the entity that is authorized to perform operations on the space (or to
  delegate authorization to other entities)

Space properties automatically added by the server:

* `url` - A relative URL to the space's description resource.
  Added by the server, used in the Space Description object as well as
  the [[[#list-spaces-operation]]] result.
* `linkset` - A relative URL to a resource which contains
  a set of links to auxiliary resources (such as to access control policy
  documents). See section [[[#space-linkset]]].
  Note that this is one of the [[[#space-level-reserved-endpoints]]].

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
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW",
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset"
}
```

#### Read Space Errors

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Space does not exist, **or** the caller has missing
  or insufficient authorization. A server MUST return the same error response in
  both cases, per [[[#error-handling]]].

### Update (or Create by Id) Space operation

When creating or modifying a Space via PUT, the client specifies the `id`
of the Space.

* Requires appropriate authorization (root zcap invoked by the space's
  controller, or a zcap granting permission to write to a particular space)
* Allows the client to update the following fields:
  - `name`
  - `controller`

#### (HTTP API) PUT `/space/{space_id}`

Note that this is a _full_ update (partial updates via http `PATCH` verb might
be supported later). However, some fields may not be updated (like `id`) and so
may be omitted from the request payload.

Note that this operation is idempotent.

* When creating a space via PUT, a `controller` property is required in the PUT request body.

Example request (creating a new space via PUT), note the lack of trailing slash:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e HTTP/1.1
Host: example.com
Accept: application/json
Content-type: application/json
Authorization: ...

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Example space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Example success response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e
```

Example request (updating the `name` property of a space). Note that
server-managed properties such as `url` and `linkset` are not client-writable
and are omitted from the request body:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e HTTP/1.1
Host: example.com
Content-type: application/json
Accept: application/json
Authorization: ...

{
  "id": "81246131-69a4-45ab-9bff-9c946b59cf2e",
  "type": ["Space"],
  "name": "Newly renamed space #1",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Space does not exist, **or** the caller has missing
  or insufficient authorization; per [[[#error-handling]]] the two are
  indistinguishable.
* [=invalid-request-body=] (400) -- the client is attempting to change an
  immutable field, such as setting a body `id` that does not match the
  `{space_id}` in the request URL.

### Delete Space operation

* Requires appropriate authorization (root zcap invoked by the space's
  controller, or a zcap granting permission to write to a particular space)
* Deletes the space and all the data (collections and resources) contained
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

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the caller has missing or insufficient authorization.
  Because DELETE is idempotent, an authorized request for an already-absent
  Space returns `204`; an under-authorized request returns `404` instead, per
  [[[#error-handling]]], so that an unauthorized caller cannot probe for
  existence.
* [=invalid-id=] (400) -- the supplied Space `id` is not URL-safe.

### List All Collections operation

* Returns the list of all Collections in a Space (that the requester has
  permission to access)
* Since Collection's `name` property is optional, default it to be the same
  value as `id` when `name` is missing. (The name is intended to drive UIs, so
  defaulting to `id` simplifies consuming client logic.)

#### (HTTP API) GET `/space/{space_id}/collections/`

Example request (note the trailing slash):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/collections/ HTTP/1.1
Host: example.com
Accept: application/json
Authorization: ...
```

Example response:

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/collections/",
  "totalItems": 2,
  "items": [
    {
      "id": "example",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/example",
      "name": "Example Collection"
    },
    {
      "id": "73WakrfVbNJBaAmhQtEeDv",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv",
      "name": "73WakrfVbNJBaAmhQtEeDv"
    }
  ]
}
```

## Collections

A collection is a namespace for Resources, and a unit of configuration, within
a space.

In other storage systems, the concept of collections has many different names.
For example, _Directory, Folder, RDBMS Table, Document Collection, Graph, WebAPI
FileList, Bucket, LDP Basic Container, EDV Vault_, and so on.

Collections do not contain other collections (this specification adopts a
flat collection structure).

<div class="note">
Rationale: Nested collections (such as those used by Solid/LDP/LWS) encode query
relationships into URL hierarchy, which creates significant complexity
(recursive permissions, recursive deletes, path traversal, depth-limited listing)
and maps poorly to flat backends like RDBMS tables or S3 buckets. The use cases
that motivate nesting (e.g., "all comments on a post") are better served by the
`query` endpoint with field-based filtering.
</div>

### Collection Data Model

Collection properties (user-writable):

* `id` - A unique collection identifier (within a given space). Created by the
  server if not provided. Note: the `{collection_id}` template parameter used in
  URL templates in this spec MUST match that collection's `id` property.
  See Appendix [[[#identifiers]]] for additional constraints.
* `type` - An array of strings, MUST include the type `Collection`. As with a
  Space's `type`, the array SHOULD be lexically sorted for a canonical, stable
  serialization.
* `name` (optional) - An arbitrary human-readable name for the collection. Does not
  have to be unique.
* `backend` (optional) - An object describing the storage backend selected for
  this collection. If not specified, defaults to the value `{ "id": "default" }`.
  The backend object's `id` property MUST be from the list of [[[#space-backends-available]]]
  for the given space. If an unavailable (unsupported) backend is specified,
  the server MUST throw an error.
  See section [[[#backends]]] for more details.

Collection properties automatically added by the server:

* `url` - A relative URL to the collection's description resource.
  Added by the server, used in the Collection Description object as well as
  the List Collections operations result.
* `linkset` - A relative URL to a resource which contains
  a set of links to auxiliary resources (such as to access control policy
  documents). See section [[[#collection-linkset]]].
  Note that this is one of the [[[#collection-level-reserved-endpoints]]].

Example collection (JSON representation):

```json
{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv",
  "type": ["Collection"],
  "name": "Verifiable Credentials Collection",
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/linkset"
}
```

### Create Collection (Add Collection to a Space) operation

When a Collection is created via a `POST`, the client can specify the `id` of
the Collection. If the `id` is not specified, one is auto-generated by the
server and returned as part of the `Location` response header.

#### (HTTP API) POST `/space/{space_id}/collections/`

Example request (`id` not specified, auto-generated by the server and returned
in the response `Location` header):

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/collections/ HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"]
}
```
In this example, the `id` in the body is not specified, and will be auto-generated
by the server. The `backend` is not specified, and will be assigned the default
value.


Example response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/21f81693-f4a3-4caa-b81c-b663d6e1e3ae
```

Example request, a valid `id` specified in the body:

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/collections/ HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "id": "credentials",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"],
  "backend": { "id": "default" }
}
```

Example response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/credentials
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=reserved-id=] (409) -- the supplied Collection `id` collides with one of the
  [[[#space-level-reserved-endpoints]]] (for example, `query` or `collections`).
* [=id-conflict=] (409) -- a Collection with the supplied `id` already exists.
  To create or replace a Collection at a client-chosen `id` without conflict,
  use the idempotent [[[#update-or-create-by-id-collection-operation]]] instead.
* [=unsupported-backend=] (409) -- the supplied `backend` id is not in that
  space's [[[#space-backends-available]]] list.

### Update (or Create By Id) Collection operation

When creating or modifying a Collection via PUT, the client specifies the `id`
of the Collection. This Collection `id` MUST NOT collide with the list of
[[[#space-level-reserved-endpoints]]].

#### (HTTP API) PUT `/space/{space_id}/{collection_id}`

Example successful "create" request (note the lack of trailing slash):

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{
  "id": "73WakrfVbNJBaAmhQtEeDv",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"],
  "backend": { "id": "default" }
}
```

```http
HTTP/1.1 201 Created
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=reserved-id=] (409) -- the supplied Collection `id` collides with one of the
  [[[#space-level-reserved-endpoints]]] (for example, `collections` or
  `linkset`).

### Get Collection Description operation

* Returns the Collection description object

#### (HTTP API) GET `/space/{space_id}/{collection_id}`

Example request (note: no trailing slash):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv HTTP/1.1
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
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv",
  "name": "Verifiable Credentials Collection",
  "type": ["Collection"],
  "linkset": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/linkset",
  "backend": { "id": "default" }
}
```

### List Collection operation

* Returns the list of Collection resources

#### (HTTP API) GET `/space/{space_id}/{collection_id}/`

Example request (note the trailing slash):

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
  "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv",
  "name": "Example JSON Documents Collection",
  "type": ["Collection"],
  "totalItems": 2,
  "items": [
    {
      "id": "321efd4e-23cb-497c-aaee-7bd26e66d39e",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/321efd4e-23cb-497c-aaee-7bd26e66d39e",
      "contentType": "application/json" 
    },
    {
      "id": "3943c87f-b617-44bc-ba75-8de2b16c3640",
      "url": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv/3943c87f-b617-44bc-ba75-8de2b16c3640",
      "contentType": "application/json" 
    }
  ]
}
```

### Delete Collection operation

#### (HTTP API) DELETE `/space/{space_id}/{collection_id}`

* Requires appropriate authorization
  - For example, when using [=zCaps=] for authorization, the request
    must either: be signed by the resource's or the space's [=controller=],
    or invoke a delegated capability that allows the `DELETE` action.

* This operation is idempotent

* (Assuming the request carries appropriate authorization) Sending a DELETE
  request to a collection that does not exist (or has already been deleted)
  results in a 204 success response

Example request (note the lack of trailing slash):

```http
DELETE /space/81246131-69a4-45ab-9bff-9c946b59cf2e/73WakrfVbNJBaAmhQtEeDv HTTP/1.1
Host: example.com
Authorization: ...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

## Resources and Blobs

### Blob Data Model

A unit of data, in transit or at rest.
(As described in the [W3C FileAPI: Blob Interface](https://w3c.github.io/FileAPI/#blob-section)).

Blob properties:

* byte stream
* `size` - length of the byte stream in bytes
  - Note: Although size can be derived from bytes, it's useful to be able to
    have it up front (for the receiving system to decide to reject an upload
    based on quota / exceeding max size, etc).
* `type` (from the IANA mime type registry) - If not specified, defaults to
  `application/octet-stream`

### Resource Data Model

A resource is a named (addressable) Blob stored in a given [=collection=],
with metadata.
The data model is derived from [W3C FileAPI: File
Interface](https://w3c.github.io/FileAPI/#file-section), but with the addition
of a few crucial properties.

In similar storage systems, a resource is called "File", "Object", "Document",
"Row", "Graph", and so on.

Resource properties:

* `id` - A unique identifier for a resource for a given collection. Created by the
  server if not provided. Note: the `{resource_id}` template parameter used in URL
  templates in this spec MUST match that resource's `id` property (as returned by
  the [[[#list-collection-operation]]]).
* `name` - (optional) - Human-readable name for the resource, useful for building
  user interfaces for browsing collection contents.
* `type` (optional) - An array of strings describing the resource. Unlike Space
  and Collection, no specific `type` value is required of a Resource.
* `url` (optional) - A relative URL (provided by the server when listing resources
  via the [[[#list-collection-operation]]])
* `contentType` (optional) - The MIME type of the default representation of the
  resource (provided by the server when listing resources
  via the [[[#list-collection-operation]]])
* Links to any metadata objects controlled by the Wallet Attached Storage server
* Links to any metadata objects modifiable by the resource's controller

### Create Resource (Add Resource to Collection) Operation

#### (HTTP API) POST `/space/{space_id}/{collection_id}/`

Example request (adds a JSON object to the `messages` collection).
Note that since no Resource id was specified, the server auto-generated an id
and returned it as part of the `Location` response header.

```http
POST /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/ HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"hi"}
```

Example success response:

```http
HTTP/1.1 201 Created
Content-type: application/json
Location: https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/6b5be748-5f39-4936-a895-409e393c399c
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=reserved-id=] (409) -- the supplied Resource `id` collides with one of the
  [[[#collection-level-reserved-endpoints]]] (for example, `query` or
  `linkset`).
* [=id-conflict=] (409) -- a Resource with the supplied `id` already exists.
  To create or replace a Resource at a client-chosen `id` without conflict,
  use the idempotent [[[#update-or-create-by-id-resource-operation]]] instead.

### Read Resource Operation

* TODO: Add language on content negotiation

#### (HTTP API) GET `/space/{space_id}/{collection_id}/{resource_id}`

* Requires appropriate authorization
  - For example, when using [zCaps](#was-authorization-profile-v0-1) for
    authorization, the request must either: be signed by the resource's or the
    space's [=controller=], or invoke a delegated capability that allows the
    [`GET` action](#get-action)

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

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the Resource does not exist, **or** the caller has
  missing or insufficient authorization; per [[[#error-handling]]] a resource
  the caller is not authorized to read is indistinguishable from one that does
  not exist.

### Update (or Create By Id) Resource Operation

When creating or modifying a Resource via PUT, the client specifies the `id`
of the Resource. This Resource `id` MUST NOT collide with the list of
[[[#collection-level-reserved-endpoints]]].

#### (HTTP API) PUT `/space/{space_id}/{collection_id}/{resource_id}`

* Requires appropriate authorization
  - For example, when using [zCaps](#was-authorization-profile-v0-1) for
    authorization, the request must either: be signed by the resource's or the
    space's [=controller=], or invoke a delegated capability that allows the
    [`PUT` action](#put-action)
* This operation is idempotent
* Returns a `204` success response

Example request to create a resource via PUT:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"hi"}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Example request to update the created resource via PUT:

```http
PUT /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: ...

{"message":"no I've changed my mind"}
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=reserved-id=] (409) -- the supplied Resource `id` collides with one of the
  [[[#collection-level-reserved-endpoints]]] (for example, `query` or
  `linkset`).
* [=not-found=] (404) -- the enclosing Space or Collection is missing or
  invalid, **or** the caller has missing or insufficient authorization; per
  [[[#error-handling]]] an under-authorized request is indistinguishable from a
  missing target.

### Delete Resource Operation

#### (HTTP API) DELETE `/space/{space_id}/{collection_id}/{resource_id}`

* Requires appropriate authorization
  - For example, when using [zCaps](#was-authorization-profile-v0-1) for authorization, the request
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
Authorization: ...
```

Example success response:

```http
HTTP/1.1 204 No Content
```

Errors (see [[[#error-type-registry]]] for canonical examples):

* [=not-found=] (404) -- the caller has missing or insufficient authorization.
  Because DELETE is idempotent, an authorized request for an already-absent
  resource returns `204`; a request that lacks sufficient authorization instead
  returns `404`, per [[[#error-handling]]], so that an unauthorized caller
  cannot probe for existence.

<section class="appendix">

## Identifiers

### Identifier Required Properties

Space, Collection and Resource identifiers used in this specification are required
to have the following properties.

1. **URL-safety** - All characters in a given identifier MUST be URL-safe.
2. **Uniqueness** - All identifiers MUST be unique within a given container.
   That is: Space `id`s (denoted by `{space_id}` in URL templates) MUST be
   unique within a given [=server=], Collection `id`s (denoted by `{collection_id}`
   in URL templates) MUST be unique within a given Space, and Resource `id`s
   (denoted by `{resource_id}` in URL templates) MUST be unique within a given
   Collection.

### Identifier Length and Format

Identifier length limits are currently left to the implementer. However,
implementations SHOULD limit identifier length to their appropriate use case.

Identifier format constraints are currently left to the implementer. Common
identifier formats include:

* Random IDs (either UUIDv4 with hyphens or using a compact encoding).
  Not recommended for _extremely_ high throughput use cases, since
  the clients or servers that are generating them might be limited by available
  device entropy.
* Semi-random Time-sortable IDs (UUIDv7). Useful for cases involving logs,
  histories, feeds, and other situations where sorting by time is desirable.
* Content-based IDs (CIDs). Useful for data deduplication use cases.
* Human-readable IDs. Some use cases might require human-readable identifiers
  (for example, a user hosting a blog in their space might want meaningful collection
  and resource IDs such as `/posts/2020-01-01-hello-world`).
</section>

<section class="appendix informative">

## Goals and Requirements

This storage specification is intended to support the following goals and
requirements.

### Local-first and Offline capable storage

Users and apps need to be able to use (provision, set up, and start reading and
writing to) storage spaces without being connected to the internet.

### Storage and sharing of public, permissioned, and private encrypted data

Although the local-first offline functionality is necessary, writing data to
stable internet-accessible URLs for the purposes of sharing them is one of the
primary use cases of this specification.

* A user needs to be able to write data (that is intended to be world-readable)
  to a cloud-accessible URL, and be able to send that URL to intended recipients
  via any out-of-band mechanism such as email, chat, and so on.

* User needs to be able to change or revoke permissions at any point after
  sharing. Note that changed permissions apply only to operations that come after
  the change (this spec is not intended to solve the general problem of DRM).

* The sharing and permission system needs to be primarily based on authorization
  capabilities (zCaps). It also needs support storage-side authorization policies
  (even if only as a way for an authorized client to receive an appropriate zcap)

* The sharing mechanism needs to be flexible and granular. For example, a given
  data resource needs to be: world-readable, or readable by groups or categories,
  or by only those possessing the required authorization capabilities, or by
  no one except the author or controller, etc

* Advanced sharing conditions are also desirable (such as "this share expires
  after X amount of time" or "this is a one-time share and will expire after
  the first successful read request")

### Stored data is opaque to the storage provider

* The spec needs to support (though not require) end-to-end client side
  encryption of the space. For plausible deniability, this might need to include
  all data (even marked as public-readable) is encrypted at rest

### Replication to user-controlled local and cloud servers

* Replication reconciles the first two requirements (data reads and writes must
  be offline-capable, but the data must eventually be able to be shared on the
  web via traditional URLs)

* Replication also provides critical availability and disaster recovery
  functionality

* Replication needs to be multi-primary (to reflect the multi-device and
  multi-client user environment)

* Multi-primary replication requires support for a versioning or conflict
  resolution mechanism

* Data, metadata, and permissions all need to be replicated

* Authorship and data provenance (the ability to tell which user or service
  created or edited a given set of data) must work in this permissioned
  multi-primary-write environment

### Serve as a General Purpose application storage backend

Intended to serve as a storage backend for credential wallets, and any other
client-side (Single Page Applications), server side, desktop, and mobile apps
and services.

### Data Portability

Data written to storage spaces using this specification needs to be portable:

* Authorized agents need to be able to export or backup all the data written,
  including all corresponding metadata and permissions

* The sharing and storage system needs to be able to support web domain
  independent identifiers. That is, a user must be able to share data at a given
  URL, then be able to migrate to a different storage service provider
  (potentially operating on a different web domain than the previous one), and
  the shared permissions to that data must not break after service migration

* While portability (and the not breaking of URLs) is relatively easy to achieve
  via redirect mechanisms (such as HTTP 301 and 302 redirect codes), this
  requires the previous service provider to be alive, available, and cooperative.
  However, this is true only of public-readable URLs, and the moment permissions
  are involved, cross-domain redirects become almost impossible to implement.
  In addition, portability from "dead servers" is also required. That is, if a
  cloud-based service provider disappears (or is otherwise unavailable), but a
  user still has a backup/export available, they should be able to set up another
  storage server (on another web domain or network address), and import/restore
  the data from backup, without shares and permissions breaking. Agents that
  the data was previously shared with must still be able to find the data at
  the new storage server location, and their permissions must still work.

### A Plurality of Data Formats and Protocols

* Spec needs to support the storage of data in any format -- binary files and
  objects, structured documents such as JSON or CBOR, contents of relational
  database tables, graphs, and anything else, all using the same unified
  metadata, sharing, and permission mechanisms.

* Storage-side schema enforcement is available but not required.

* Spec needs to be able to support multiple protocols and APIs, such as HTTP,
  JSON-RPC, DIDComm, local client APIs, and more.

### Permissioned Query and Search functionality

* Where appropriate (such as for unstructured text, structured documents, RDBMSs
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

* Implementations of this spec need to be usable with FIPS-compliant
  cryptographic algorithms

### Anti-Goals

#### Use cases do not include "zero trust" environments

This storage specification is intentionally positioned to not be used in
"zero trust" environments, which in practice means the usage of untrusted sync
and replication nodes while solely relying on encryption as the authorization
mechanism.

To put it a different way -- all encryption has an unpredictable half-life, and
some use cases do not permit relying on encryption only for access control.
Instead, a _combination_ of encryption and authorization enforcement by
minimally trusted storage servers is required.

</section>

<section class="appendix">

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
   via an HTTP GET), use the simple `{ "type": "PublicCanRead" }` WAS
   Authorization syntax, see below.
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
5. Access Control / Policy language data model: see
   [[[#access-control-policies]]] (`PublicCanRead` is the only normative type for
   v0.1)

### Space `controller` and the Root of Trust

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

Space `controller`s MUST be in the form of a [DID](https://www.w3.org/TR/did-1.0/).

For minimal compatibility, all WAS implementations MUST support the
[`did:key` DID Method](https://w3c-ccg.github.io/did-key-spec/), using the
Multikey encoding of `Ed25519` elliptic curve keys, as specified in the
[Multikey section of the CID spec](https://www.w3.org/TR/cid-1.0/#Multikey)
as the space `controller`.

When a space is created via an HTTP [POST](#http-api-post-spaces) or
[PUT](#http-api-put-space-space_id) operation, the controller for that space
is set explicitly. That is, a client specifies the `controller` as part of the
payload of the PUT or POST create space request, and the server MUST check that
the methods(key IDs) used in the headers are authorized in the `capabilityInvocation`
section of the `controller`'s DID document.

See below in the [HTTP POST](#http-api-post-spaces) sections for examples of
`controller` determination and verification.

### Performing Authorized API Calls

Unless otherwise explicitly allowed via access control policy (see below),
all W.A.S. API calls require authorization.

This can be done in one of two ways:

1. (for admin-like root access) Use the `controller` DID directly to sign
   HTTP API requests using the HTTP Signatures specification, invoking the
   target's [=root capability=].
2. (for advanced delegatable use cases) Use HTTP Signatures in combination
   with [Authorization Capabilities v0.3](https://w3c-ccg.github.io/zcap-spec/),
   and include a capability invocation header in the API request.

### Authorization Actions and the Root Capability

A capability invocation names an [=action=] that the invoked capability must
permit. WAS uses the uppercase HTTP method names as its action vocabulary:

* <dfn id="get-action">`GET`</dfn> -- read a Space, Collection, or Resource. A
  `HEAD` request is authorized as a `GET`.
* <dfn id="post-action">`POST`</dfn> -- create a child item in a container (add a
  Resource to a Collection, a Collection to a Space, or a Space to the Spaces
  Repository).
* <dfn id="put-action">`PUT`</dfn> -- create-by-id or replace a Space,
  Collection, or Resource.
* <dfn id="delete-action">`DELETE`</dfn> -- delete a Space, Collection, or
  Resource.

A request is authorized by a capability when all the following hold:

1. the capability's `invocationTarget` matches the request's [=target=] -- the
   full request URL (scheme, host, port, and path);
2. the capability's `allowedAction` includes the request's action (the HTTP
   method); and
3. the invocation is signed by a key the capability authorizes, carried as a
   valid HTTP Signature over the request (see
   [[[#performing-authorized-api-calls]]]).

#### Root Capability

Every [=target=] has an implied **root capability** whose `controller` is the
Space's [=controller=]. It is identified by the URI `urn:zcap:root:` followed by
the percent-encoded target URL:

```json
{
  "@context": "https://w3id.org/zcap/v1",
  "id": "urn:zcap:root:https%3A%2F%2Fexample.com%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e%2Fmessages%2Fhello-world",
  "invocationTarget": "https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/hello-world",
  "controller": "did:key:z6MkpBMbMaRSv5nsgifRAwEKvHHoiKDMhiAHShTFNmkJNdVW"
}
```

The Space [=controller=] MAY invoke the root capability directly -- signing the
request with a key listed in the `capabilityInvocation` section of the
controller's DID document -- to perform any operation. This is the "root access"
path. All other authorized access derives from a capability delegated, directly
or transitively, from this root.

#### Delegation

To grant another agent access, the [=controller=] (or any agent holding a
sufficiently broad capability) delegates a capability that names the grantee as
its new `controller`, the `invocationTarget` to scope it to, and the
`allowedAction`s to permit. A delegation MAY set an `expires` time. For example,
granting another DID read-only access to a single Collection:

```json
{
  "@context": "https://w3id.org/zcap/v1",
  "id": "urn:uuid:6c9f3a1e-2b4d-4f8a-9c1e-7d2b3a4c5e6f",
  "parentCapability": "urn:zcap:root:https%3A%2F%2Fexample.com%2Fspace%2F81246131-69a4-45ab-9bff-9c946b59cf2e%2Fmessages%2F",
  "invocationTarget": "https://example.com/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/",
  "controller": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
  "allowedAction": ["GET"],
  "expires": "2026-12-31T23:59:59Z",
  "proof": { "...": "delegation proof signed by the parent capability's controller" }
}
```

The delegated capability is handed to the recipient out of band. The recipient
invokes it by signing a request with their own key and including the capability
in the `Capability-Invocation` header.

### Specifying Access Policy With Space Link Sets

To set access control policy for a space, use the `linkset` property.

Example (fetching a space's link set):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset HTTP/1.1
Host: example.com
Accept: application/linkset+json
Authorization: ...
```

Response:

```http
HTTP/1.1 200 OK
Content-type: application/linkset+json

{
  "linkset": [
    {
      "anchor": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
      "https://wallet.storage/spec#policy": [
        { 
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/policy",
          "type": "application/json"
        }
      ]
    }
  ]
}
```

Example (fetching a specific policy document from the link set):

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/policy HTTP/1.1
Host: example.com
Authorization: ...
```

```http
HTTP/1.1 200 OK
Content-type: application/json

{ "type": "PublicCanRead" }
```

### Access Control Policies

Capabilities answer the question "does the caller hold a credential that grants
this action?" Access control *policies* answer the complementary question
"does this target grant this action to callers in general (or to a named set of
principals)?" Policies are how a [=controller=] makes a target public-readable
(or, in future profiles, shares it with a list of people or groups) without
having to issue a capability to each caller.

A policy is a JSON document with a required `type` property, stored at the
[=policy=] auxiliary resource of a Space, Collection, or Resource and
discoverable via the [=policy=] linkset relation (see [[[#space-linkset]]] and
[[[#collection-linkset]]]).

**Evaluation contract:**

* **Capability first, policy second.** A request is first checked against any
  capability invocation it carries. The effective policy is consulted only as a
  fallback, and it can only **broaden** access -- a policy never denies a caller
  who presents a valid capability.
* **Fail-closed.** An absent policy, or a policy whose `type` an implementation
  does not recognize, grants nothing.
* **Most-specific-wins inheritance.** The effective policy for a target is the
  one set at the most specific level that has a policy document: a Resource
  policy overrides a Collection policy, which overrides a Space policy.
* **Access kind.** For policy evaluation, the request action is reduced to a
  coarse access kind: [=GET=] (and `HEAD`) is a `read`; [=POST=], [=PUT=], and
  [=DELETE=] are a `write`.

#### `PublicCanRead`

For v0.1, the only normative policy `type` is `PublicCanRead`:

```json
{ "type": "PublicCanRead" }
```

It grants the `read` access kind to any caller (including unauthenticated ones)
and grants no write access. This is the canonical "public read" pattern -- for
example, hosting an HTML file or an image that anyone may `GET`, while writes
still require a capability. Setting it on a Space makes the whole Space
public-readable (subject to any more specific Collection or Resource policy);
setting it on a single Resource exposes only that Resource.
</section>

<section class="appendix">

## Linksets

Linksets (from [[RFC9264]]) serve as the main feature detection and extension
mechanism. They can be discovered, via the `linkset` property, from the following:

* The Space Description object, see [[[#space-data-model]]].
* The Collection Description object, see [[[#collection-data-model]]].

### Space Linkset

The space `linkset` resource (one of the [[[#space-level-reserved-endpoints]]]),
located at `/space/{space_id}/linkset` contains a set of links to auxiliary
resources and extension points:

* <dfn id="policy">`policy`</dfn> (`https://wallet.storage/spec#policy`) - A link to the
  `/space/{space_id}/policy` resource, which contains a set of links to access control
  policy documents.
* <dfn id="backends-available">`backends-available`</dfn>
  (`https://wallet.storage/spec#backends-available`) - A link to the
  `/space/{space_id}/backends` "Backends Available" resource.
* `/space/{space_id}/query` - Reserved for cross-space query operations.

Example space linkset resource request and response:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/linkset HTTP/1.1
Accept: application/linkset+json
Authorization: ...
```

Response:

```http
HTTP/1.1 200 OK
Content-type: application/linkset+json

{
  "linkset": [
    {
      "anchor": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/",
      "https://wallet.storage/spec#policy": [
        { 
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/policy",
          "type": "application/json"
        }
      ],
      "https://wallet.storage/spec#backends-available": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/backends",
          "type": "application/json"
        }
      ]
    }
  ]
}
```

### Collection Linkset

The collection `linkset` resource (one of the [[[#collection-level-reserved-endpoints]]]),
located at `/space/{space_id}/{collection_id}/linkset` contains a set of links
to auxiliary resources and extension points:

* The [=policy=] relation (`https://wallet.storage/spec#policy`) - A link to the
  `/space/{space_id}/{collection_id}/policy` resource, which contains a set of links
  to access control policy documents.
* <dfn id="backend">`backend`</dfn> (`https://wallet.storage/spec#backend`) - A link to the
  detailed `/space/{space_id}/{collection_id}/backend` "Backend Selected for this
  collection" resource.
* `/space/{space_id}/{collection_id}/query` - (Optional) Reserved for query
  operations within a collection.

Example collection linkset resource request and response:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/linkset HTTP/1.1
Accept: application/linkset+json
Authorization: ...
```

Response:

```http
HTTP/1.1 200 OK
Content-type: application/linkset+json

{
  "linkset": [
    {
      "anchor": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/",
      "https://wallet.storage/spec#policy": [
        { 
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/policy",
          "type": "application/json"
        }
      ],
      "https://wallet.storage/spec#backend": [
        {
          "href": "/space/81246131-69a4-45ab-9bff-9c946b59cf2e/messages/backend",
          "type": "application/json"
        }
      ]
    }
  ]
}
```

</section>

<section class="appendix">

## Backends

Backends are an **optional** infrastructure concern that is orthogonal to the
hierarchical Spaces Repository > Space > Collection > Resource storage model.

Available backends are registered on the Space level, as a combination of
server-side configuration and client-side "Bring Your Own Storage" registration.

For example, on the server configuration side, a given server might support several
backends -- a file system default backend, an EDV encrypted backend, and a
PostgreSQL database backend. And on the client side, a user might register an
external storage provider by connecting to their Dropbox account.

When a Collection is created, the client can optionally specify the preferred
backend for that Collection. If no preferred backend is specified, one is assigned
by the server (usually the `default` backend).

Backends are an **optional system** that exists to serve advanced use cases that
need fine-grained control over storage configurations.
An implementer or client of a given server can omit the `backend` property when
creating a Collection. By default, if not specified, all Collections are
assigned the `default` backend.

### Space Backends Available

Example request:

```http
GET /space/81246131-69a4-45ab-9bff-9c946b59cf2e/backends HTTP/1.1
Accept: application/json
Authorization: ...
```

Example success response:

```http
HTTP/1.1 200 OK
Content-type: application/json

[
  { "id": "default" },
  { "id": "dropbox" },
  { "id": "edv" }
]
```

### Collection Backend Selected

Each collection has an optional `backend` property that is set during its creation
(see [[[#collection-data-model]]]). If not specified, it is assumed to have the
`id` of `default`.

### Quotas

TODO: Add quota reporting semantics. Some Backends support quota limit
enforcement; where supported, a per-Space quota report (grouped by backend) is
available at `GET /space/{space_id}/quotas`, and a per-Collection report at
`GET /space/{space_id}/{collection_id}/quota` (not all Backends support
per-collection quotas).

</section>

<section class="appendix">

## Reserved Path Segment Registry

### Space-level reserved endpoints

The following path segments represent reserved API endpoints for [[[#spaces]]]
level operations. Usually, the path segment following the `/space/{space_id}/` prefix
is the id of a Collection. The list of reserved endpoints below means that
collections `id`s MUST NOT collide with the corresponding reserved segments.

| Reserved API Endpoint           | Reserved segment | Purpose                                |
|---------------------------------|------------------|----------------------------------------|
| `/space/{space_id}/policy`      | `policy`         | Access control policy                  |
| `/space/{space_id}/backends`    | `backends`       | Storage backends available             |
| `/space/{space_id}/collections` | `collections`    | List and create collections            |
| `/space/{space_id}/export`      | `export`         | Export (download) space contents       |
| `/space/{space_id}/linkset`     | `linkset`        | Links to auxiliary resources           |
| `/space/{space_id}/query`       | `query`          | Reserved for cross-collection queries  |
| `/space/{space_id}/quotas`      | `quotas`         | Reserved for per-backend quota report  |

If a client attempts to create a collection with an `id` that collides with a
reserved segment list above, the server MUST return a 409 Conflict error.

### Collection-level reserved endpoints

The following path segments represent reserved API endpoints for [[[#collections]]]
level operations. Usually, the path segment following the
`/space/{space_id}/{collection_id}/` prefix is the id of a Resource. The list of
reserved endpoints below means that resource `id`s MUST NOT collide with the
corresponding reserved segments.

| Reserved API Endpoint                        | Reserved segment | Purpose                             |
|----------------------------------------------|------------------|-------------------------------------|
| `/space/{space_id}/{collection_id}/policy`   | `policy`         | Access control policy               |
| `/space/{space_id}/{collection_id}/backend`  | `backend`        | Storage backend selected            |
| `/space/{space_id}/{collection_id}/linkset`  | `linkset`        | Links to auxiliary resources        |
| `/space/{space_id}/{collection_id}/query`    | `query`          | Query resources within a collection |
| `/space/{space_id}/{collection_id}/quota`    | `quota`          | Storage quota report for collection |

If a client attempts to create a resource with an `id` that collides with a
reserved segment list above, the server MUST return a 409 Conflict error.

### Resource-level reserved endpoints

The following path segments represent reserved API endpoints for Resource
level operations.

| Reserved API Endpoint                                  | Reserved segment | Purpose                              |
|--------------------------------------------------------|------------------|--------------------------------------|
| `/space/{space_id}/{collection_id}/{resource_id}/meta` | `meta`           | User-defined metadata for a resource |

</section>

<section class="appendix">

## Error Type Registry

This specification uses [[RFC9457]] Problem Details for HTTP APIs for error
responses (see [[[#error-handling]]]). The `type` property of a problem
response is a URI identifying the _kind_ of problem. Following [[RFC9457]],
a `type` is reused across operations: a single kind such as
[=invalid-id=] is emitted by Create, Update, and Read operations
across Spaces, Collections, and Resources alike. The per-occurrence specifics
belong in the `errors` array (`detail` and an optional `pointer`), and the
short human-readable summary in `title`.

Each `type` URI is a fragment anchor into this registry. The status codes
listed below are typical; a single kind MAY be returned with more than one
status code depending on the operation.

| `type` URI | Anchor | Typical status | Description |
|------------|--------|----------------|-------------|
| `https://wallet.storage/spec#not-found` | <dfn id="not-found">not-found</dfn> | 404 | The resource (Space, Collection, or Resource) does not exist, **or** the caller is not authorized to access it. These two conditions are deliberately indistinguishable -- see the privacy note below. |
| `https://wallet.storage/spec#invalid-id` | <dfn id="invalid-id">invalid-id</dfn> | 400 | A Space, Collection, or Resource `id` is missing or not URL-safe. |
| `https://wallet.storage/spec#reserved-id` | <dfn id="reserved-id">reserved-id</dfn> | 409 | A client-supplied `id` collides with a [[[#reserved-path-segment-registry]]] segment. |
| `https://wallet.storage/spec#id-conflict` | <dfn id="id-conflict">id-conflict</dfn> | 409 | A client-supplied `id` in a `POST` create operation already exists. (Create-or-replace by `id` is done idempotently via `PUT`, which does not conflict.) |
| `https://wallet.storage/spec#invalid-request-body` | <dfn id="invalid-request-body">invalid-request-body</dfn> | 400 | The request body is missing or invalid (e.g. a required property is absent). Entries in `errors` SHOULD carry a `pointer` to the offending field. |
| `https://wallet.storage/spec#missing-content-type` | <dfn id="missing-content-type">missing-content-type</dfn> | 400 | A required `Content-Type` header is missing. |
| `https://wallet.storage/spec#missing-authorization` | <dfn id="missing-authorization">missing-authorization</dfn> | 401 | Required `Authorization` / `Capability-Invocation` headers (or proof of possession) are missing. |
| `https://wallet.storage/spec#invalid-authorization-header` | <dfn id="invalid-authorization-header">invalid-authorization-header</dfn> | 400 | An `Authorization`, `Capability-Invocation`, or `Digest` header is malformed, unparseable, or failed verification. |
| `https://wallet.storage/spec#controller-mismatch` | <dfn id="controller-mismatch">controller-mismatch</dfn> | 400 | The DID that signed the capability invocation does not match the `controller` supplied in a Create Space request body. |
| `https://wallet.storage/spec#unsupported-backend` | <dfn id="unsupported-backend">unsupported-backend</dfn> | 409 | A requested `backend` id is not in the space's [[[#space-backends-available]]] list. |
| `https://wallet.storage/spec#invalid-import` | <dfn id="invalid-import">invalid-import</dfn> | 400 | An uploaded archive is not a valid WAS space export. |
| `https://wallet.storage/spec#storage-error` | <dfn id="storage-error">storage-error</dfn> | 500 | An underlying storage operation failed. |
| `https://wallet.storage/spec#internal-error` | <dfn id="internal-error">internal-error</dfn> | 500 | An unexpected server-side fault with no more specific kind. |

**Privacy: the `not-found` kind is intentionally merged.** Under the principle
of maximum privacy (see [[[#error-handling]]]), an unauthorized client MUST NOT
be able to discover the existence of a Space, Collection, or Resource from an
error response. The [=not-found=] kind therefore covers both
"resource absent" and "invalid authorization"; implementations MUST NOT split
it into distinguishable `type` values, and MUST NOT otherwise let the response
(status code, `title`, or `detail`) reveal whether the resource exists.

This merging applies to "insufficient-authorization" and "absent-authorization"
against an existing target. It does not apply to request- or
credential-validation failures, which describe the request rather than the
target and so MAY use their own precise `type`s -- see [[[#error-handling]]].

### Error Response Examples

Canonical example responses for the error kinds defined above, referenced by
the per-operation "Errors" lists throughout this specification. The `title`
strings are illustrative, not normative: for example, the noun in a
[=not-found=] `title` varies with the target (Space, Collection, or Resource),
and a `title` may name the operation that failed. Kinds not shown here
([=missing-content-type=], [=invalid-authorization-header=],
[=invalid-import=], [=storage-error=], [=internal-error=]) follow the same
shape; they will gain examples as their corresponding sections are drafted.

[=not-found=] -- a missing target, or missing or insufficient authorization
(deliberately indistinguishable, see the privacy note above):

```http
HTTP/1.1 404 Not Found
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#not-found",
  "title": "Resource not found or insufficient authorization."
}
```

[=invalid-id=] -- an `id` that is not URL-safe (see [[[#identifiers]]]):

```http
HTTP/1.1 400 Bad Request
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#invalid-id",
  "title": "Invalid Space id.",
  "errors": [
    {
      "detail": "Space 'id' must be URL-safe.",
      "pointer": "#/id"
    }
  ]
}
```

[=reserved-id=] -- a client-supplied `id` that collides with a segment from the
[[[#reserved-path-segment-registry]]]:

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#reserved-id",
  "title": "Invalid collection id (from reserved list)."
}
```

[=id-conflict=] -- a `POST` create operation supplying an `id` that already
exists:

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#id-conflict",
  "title": "A Collection with this id already exists.",
  "errors": [
    {
      "detail": "Use PUT to create-or-replace a Collection at a chosen id.",
      "pointer": "#/id"
    }
  ]
}
```

[=invalid-request-body=] -- a missing or invalid request body (here, a Create
Space request without the required `controller` property):

```http
HTTP/1.1 400 Bad Request
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#invalid-request-body",
  "title": "Invalid Create Space body.",
  "errors": [
    {
      "detail": "'controller' property is required.",
      "pointer": "#/controller"
    }
  ]
}
```

[=missing-authorization=] -- required authorization (here, a proof of
possession signature on a Create Space request) is missing:

```http
HTTP/1.1 401 Unauthorized
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#missing-authorization",
  "title": "Invalid Create Space request.",
  "errors": [
    {
      "detail": "Valid proof of possession of the 'controller' DID must be provided."
    }
  ]
}
```

[=controller-mismatch=] -- the signing DID in the `Authorization` header does
not match the DID specified in the `controller` property of a Create Space
request body:

```http
HTTP/1.1 400 Bad Request
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#controller-mismatch",
  "title": "Invalid Create Space request.",
  "errors": [
    {
      "detail": "The signing DID from the Authorization header must match the 'controller' DID in request body.",
      "pointer": "#/controller"
    }
  ]
}
```

[=unsupported-backend=] -- a `backend` id that is not part of that space's
[[[#space-backends-available]]] list:

```http
HTTP/1.1 409 Conflict
Content-type: application/problem+json

{
  "type": "https://wallet.storage/spec#unsupported-backend",
  "title": "Unsupported backend id, check the space's 'backends available' list."
}
```

</section>

<section class="appendix informative">

## IANA Considerations

This section will be submitted to the Internet Engineering Steering Group (IESG)
for review, approval, and registration with IANA.
</section>

