# API Design Notes

The contract is `inventory-api.yaml` (OpenAPI 3.1). It passes the Redocly linter with no errors. The two warnings are about placeholder server addresses. These notes explain the choices so you can talk through them.

## Operations

| Operation | Method and path | Source |
|---|---|---|
| Search stock items | `GET /v1/stock-items` | Search index |
| Get one stock item | `GET /v1/stock-items/{stockItemId}` | Database |
| Receive stock, one or a batch | `POST /v1/stock-items` | Database |
| Change allowed fields | `PATCH /v1/stock-items/{stockItemId}` | Database |
| List movements | `GET /v1/stock-items/{stockItemId}/movements` | Database |
| Search parts, get one, create | `GET /v1/parts`, `GET /v1/parts/{partId}`, `POST /v1/parts` | Index, database, database |
| Search and create locations | `GET /v1/locations`, `POST /v1/locations` | Index, database |
| Check a background operation | `GET /v1/operations/{operationId}` | Database |

I added `PATCH` for stock items and the operation resource, because the assignment needs both concurrency control on updates and a 202 path.

Identifiers in URLs are the public UUIDs, never the database numbers.

## Search

- **Free text** in `q`. Part numbers match however they are typed. The client sends what the user typed. No search-engine syntax is accepted or shown.
- **Filters** are named fields (`conditionCode`, `isServiceable`, `shelfLifeExpiresBefore`). Several values are comma separated. The server turns them into index queries.
- **Facets** are requested with `facets=conditionCode,statusCode` and returned as value and count pairs, computed with the other filters applied.
- **Sorting** is `sort=-receivedAt,partNumber`, limited to a listed set of fields. Anything else answers 400 with `SORT_FIELD_NOT_ALLOWED`.
- **Sparse fields** with `fields=partNumber,quantityOnHand`. Identifiers always come back.
- **Paging** is cursor-based only. The cursor is opaque and works at any depth, since the server uses a sort-key cursor. There is no offset, which fails past a few thousand rows.

## Create

- **Single item** returns `201` with the new object as the body, a `Location` header, and an `ETag`.
- **Batch of 2 to 500** returns `202` with an operation resource and a `Location` to poll. Each item succeeds or fails on its own, so one bad row never blocks the rest. The final state is `SUCCEEDED`, `PARTIALLY_SUCCEEDED` or `FAILED`, with one result per item in the order sent.
- **Idempotency.** The `Idempotency-Key` header is required. The server saves the key, a hash of the body and the result for 24 hours. The same key with the same body returns the saved result, marked `Idempotency-Replayed: true`. The same key with a different body is `422 IDEMPOTENCY_KEY_REUSED`. A repeat that arrives while the first is still running is `409 IDEMPOTENCY_REQUEST_IN_PROGRESS`. In a batch, the key covers the whole batch.
- **Validation** is listed in the operation text and matches the schema rules: serialised items have quantity 1, serial number unique per part, certificate required if the part says so, shelf life and manufacture date likewise, serviceable and location rules, and money with its currency. All broken rules are returned together as field errors.

## Concurrency

- Every read of one object returns an `ETag`, which holds the `rowVersion`.
- An update sends `If-Match`. Without it, the answer is `428`.
- If two storekeepers open the same item, the first save wins. The second gets `412 VERSION_CONFLICT` with the current version number. The client reads the item again, shows what changed, and lets the user retry. The server never merges silently.
- The database update is `WHERE id = ? AND row_version = ?`, so this holds even when requests arrive together.
- Quantity, location, condition and ownership do not change through `PATCH`. They change through movements, so the ledger stays the single record.

## Consistency

Search follows the database by a couple of seconds, so the item you just created may not be in the results. The contract handles this three ways:

1. **Say how fresh it is.** Every search response has a `freshness` block with `indexedThrough` and `lagMilliseconds`. The screen can show "updated 2 seconds ago" and warn if the lag grows.
2. **Let the client wait.** Writes return `X-Consistency-Token`. The client sends it back in a `Consistency-Token` header on its next search. The server waits up to 2 seconds for the index to include that write, then answers. `includesConsistencyToken` says whether it did. This gives read-your-own-writes for the common "receive, then list" flow.
3. **Offer a current answer.** `GET` by id always reads the database. The Angular app, after a create, uses the object from the `201` and opens it by id.

## Errors

- Every error is `application/problem+json` following RFC 9457, with `type`, `title`, `status`, `detail`, `instance`, plus `code`, `requestId` and an `errors` list. Each field error has a JSON Pointer and a stable code.
- The code catalogue is the `ErrorCode` list in the contract. Codes are never renamed or reused. Clients treat unknown codes as a general error of that status, so new codes can be added without breaking anyone.
- Not found and not allowed at another base both answer `404`, so the existence of other bases' data is not revealed.

## Security

- OAuth 2.0 authorisation code with PKCE for the Angular app, and client credentials for other modules. OpenID Connect handles sign-in and discovery.
- Six scopes: stock, part and location, each with read and write.
- The token carries a `bases` claim. The server adds `base_id` to every database query and every search filter from that claim, and it intersects any `baseId` the client sends. A client can narrow but never widen. A base outside the claim answers `403` on an explicit request and silently disappears from searches.
- Scopes say what you may do. Bases say where. Both are checked on every request.

## Reuse

Shared components: `Quantity` (value and unit), `Money` (string amount and currency), `Page`, `Facets`, `Freshness`, `Problem`, `FieldError`, `ErrorCode`, and the `MovementTypeCode` and `CertificateTypeCode` lists. Condition, ownership and status are strings from maintained code lists, not enums, so a new code in a lookup table does not need a contract change.

The worked example is in `POST /v1/stock-items`: receiving a serialised component with an EASA Form 1 certificate and a life entry, with the matching `201` response.

## Delivery

| Step | Approach |
|---|---|
| Contract first | The YAML file lives in the repository and changes through pull requests. Code follows the contract, not the other way round. |
| .NET server | Generate controller interfaces and models from the contract (NSwag or a similar generator). Developers implement the interfaces, so the server cannot drift from the contract without a compile error. |
| Angular client | Generate the TypeScript client and types from the same file (OpenAPI Generator), so field names match without hand-copying. |
| Linting | Redocly or Spectral in the build, with rules for naming (lowerCamelCase fields), required descriptions, and examples. A breaking-change check compares each pull request with the released contract and fails on removed fields, new required fields, or changed types. |
| Does the implementation match? | Contract tests run in the build. Schemathesis or a similar tool sends generated requests to a running test instance and checks every response against the contract. Hand-written scenario tests cover idempotency replay, `412` on stale writes, base filtering, and the consistency token. |
| Versioning | `/v1` in the path. Adding optional fields is not a breaking change. Removing or changing fields means `/v2`, with the old version kept for an agreed period. |

## Assumptions and open questions

1. Rate limits are returned as `429`, but the numbers are not set.
2. Idempotency keys are kept for 24 hours and operations for 7 days. Both are assumptions.
3. Stock item `PATCH` only covers allocation block, remarks and barcode. Other changes are movements, which are separate commands outside the six operations the assignment requires. They would be added in the same style.
4. A callback URL for batches is optional. Polling always works. It is a convenience, and the security of callbacks would need review.
5. Condition, ownership and status codes are open strings. A lookup endpoint to list them is not in this version.
