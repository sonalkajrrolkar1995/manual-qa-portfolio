# Section 6 - API Analysis

No automation. Analysis based on manual testing and response observation.

---

## ReqRes

### Endpoint Behavior

**GET /api/users?page=n**

Returns a paginated list. Response envelope: `page`, `per_page`, `total`, `total_pages`, `data` (array), `support` (object).

- `per_page` is always 6 regardless of query parameters. No `per_page` override is supported.
- `total` is always 12. `total_pages` is always 2. These values do not change even when the data array is empty (e.g. page 3).
- The `support` object contains a URL and a message that appears to be promotional content from Sauce Labs. It is returned on every list response and has no functional purpose in the API contract.
- Page 0 and negative page values return page 1 data. The `page` field in the response mirrors the invalid input rather than correcting to 1.

**GET /api/users/:id**

Returns a single user wrapped in `{ "data": {...}, "support": {...} }`.

- IDs 1-12 return 200 with user data.
- ID 13 and above return 404 with `{}` - an empty JSON body, not a structured error.
- The inconsistency: list endpoints return a `support` wrapper on 200, but error responses return `{}` with no error field, no message, and no code.

**POST /api/users**

Accepts any JSON body. Returns 201 with `id` (string), `createdAt` (ISO timestamp), and any fields that were sent.

- No field validation. Empty body returns `{ "id": "...", "createdAt": "..." }`.
- The `id` returned is a high integer (e.g. 742, 856). These IDs do not exist in the seeded dataset - a GET on the returned ID will 404.
- `createdAt` is the actual current timestamp, not mocked. This is the only truly dynamic field in the API.
- Fields sent in the request are echoed back as-is. Whitespace, numbers, special characters - all accepted and echoed.

**PUT /api/users/:id**

Returns 200 with the sent fields plus `updatedAt`. Does not return a full user object - only the fields you sent plus the timestamp.

- The response implies the resource was updated. In reality, a subsequent GET returns the original seeded data unchanged.
- Works for any ID including non-existent ones. PUT to `/api/users/99999` returns 200 + updatedAt.

**PATCH /api/users/:id**

Same behavior as PUT from a response perspective. Returns 200 with sent fields plus `updatedAt`. Same caveat: nothing actually changes. Same lack of ID validation.

**DELETE /api/users/:id**

Returns 204 for all requests regardless of whether the ID exists.

- No response body.
- No distinction between deleting ID 1 (which exists) and ID 99999 (which does not).

**POST /api/login**

Accepts `email` and `password`. Returns `{ "token": "QpwL5tpe83ilfN2" }` on success.

- Token is always the same static value. It does not change between requests or between users.
- Missing password returns 400 with `{ "error": "Missing password" }`. This is the only endpoint with real input validation.
- Invalid credentials return 400 with `{ "error": "user not found" }`. Only specific email addresses (the seeded ones) return a token.
- The token has no expiry, no scope, and is not used to authorize any other endpoint.

**POST /api/register**

Same structure as login. Works only for seeded email addresses. Returns `{ "id": n, "token": "..." }`.

---

### Validation Gaps

| Endpoint | Gap | Impact |
|---|---|---|
| POST /api/users | No field validation | Any data accepted - teaches incorrect API contract assumptions |
| GET /api/users?page=n | No boundary validation on page parameter | Silent bad behavior on page 0/-1 |
| DELETE /api/users/:id | No 404 for non-existent resources | Delete always looks successful |
| PUT/PATCH /api/users/:id | Any ID accepted | 200 response for updates to non-existent resources |
| GET /api/users/:id (404) | Error response is `{}` | No structured error format for consumers to parse |

---

### Response Structure Inconsistencies

- 200 responses: `{ "data": {...}, "support": {...} }`
- 404 responses: `{}`
- 400 responses: `{ "error": "message" }`
- 201 responses: flat object, no `data` wrapper

There is no consistent error envelope. Consumers must handle four different response structures depending on status code.

---

### Negative Scenarios Worth Testing

These scenarios should be tested before building any integration against this API:

1. `GET /api/users?page=0` - verify whether behavior is defined in consumer pagination logic
2. `DELETE /api/users/:id` on a just-created ID (not seeded) - confirm 204 matches the behavior for seeded IDs
3. `POST /api/users` with `Content-Type: text/plain` - verify how the API handles wrong content type
4. `GET /api/users/0` - ID 0 - verify whether the API treats this as invalid or returns 404
5. `POST /api/login` with the email of a seeded user but wrong password - verify the 400 behavior vs. "user not found"

---

## Restful Booker

### Endpoint Behavior

**POST /auth**

Accepts `{ "username": "...", "password": "..." }`. Returns `{ "token": "abc123..." }` on success.

- Success: HTTP 200 with token string.
- Failure: HTTP 200 with `{ "reason": "Bad credentials" }`. HTTP status is 200 in both cases.
- Token format appears to be a short alphanumeric string, not a JWT. It cannot be decoded.
- Token expires - observed TTL is approximately 10 minutes but this is undocumented.
- Getting a new token does not invalidate the old one.

**GET /booking**

Returns an array of objects: `[{ "bookingid": n }, ...]`.

- No authentication required.
- Accepts filter parameters: `firstname`, `lastname`, `checkin`, `checkout`.
- No pagination. Returns all matching bookings in one response.
- Filter behavior: exact match only. Partial name matches do not return results.
- Empty filter (no params): returns all booking IDs in the system.
- Unknown filter parameter: ignored silently, returns all bookings.

**POST /booking**

Creates a booking. Returns `{ "bookingid": n, "booking": { full booking object } }`.

- No authentication required for creation. Anyone can create a booking.
- Required fields: not formally documented. The API accepts bookings with missing fields in some cases.
- `totalprice`: non-numeric values coerced to null silently. Negative values accepted.
- `depositpaid`: must be boolean. String "true" is not coerced to boolean - behavior varies.
- `bookingdates.checkin` and `bookingdates.checkout`: date strings in YYYY-MM-DD format. No validation of date order.
- `additionalneeds`: free text. No length limit observed.
- Response contains the created `bookingid` and a full copy of the booking as stored - useful for verifying what was actually saved vs. what was sent.

**GET /booking/:id**

Returns the booking object directly - not wrapped.

- No authentication required.
- Non-existent ID: returns 404 with `"null"` as the body (the string null, not JSON null).
- The inconsistency: other endpoints return valid JSON on error. This one returns a non-JSON string.

**PUT /booking/:id**

Full replacement. Requires all fields to be present.

- Requires auth via `Cookie: token=<value>` or `Authorization: Basic <base64>`.
- No auth: 403.
- Invalid token: 403.
- Partial body (missing fields): returns 400 in most cases but behavior is inconsistent.
- Returns the updated booking object on success (200).

**PATCH /booking/:id**

Partial update. Accepts any subset of booking fields.

- Same auth requirements as PUT.
- Empty body `{}`: returns 200, no changes made, no error.
- Returns the full updated booking object on success (200).
- A PATCH that sends only `firstname` will update only `firstname` and leave all other fields unchanged - this is correct partial update behavior.

**DELETE /booking/:id**

- Requires auth.
- Returns 201 on success (should be 204).
- No response body.
- Deleting a non-existent ID: returns 405 (Method Not Allowed) in some observed cases, rather than 404. Behavior is inconsistent.
- After deletion, GET on the same ID returns 404.

---

### Validation Gaps

| Field | Gap |
|---|---|
| `bookingdates.checkin` / `checkout` | No date order validation. checkout can precede checkin. |
| `totalprice` | Accepts negative values. Coerces non-numeric to null silently. |
| `depositpaid` | String "true" behavior is undefined. |
| PATCH with `{}` | Returns 200 for a no-op update - no error for empty update. |
| POST /auth failure | Returns 200 instead of 401. |
| DELETE non-existent ID | Inconsistent status: 405 observed in some cases instead of 404. |

---

### Response Structure Inconsistencies

- `GET /booking/:id` success: flat booking object
- `GET /booking/:id` 404: `"null"` (string, not JSON)
- `POST /booking` success: `{ "bookingid": n, "booking": {...} }`
- `DELETE /booking/:id`: 201 with empty body
- `POST /auth` failure: 200 with `{ "reason": "..." }`

There is no consistent error format. Error responses vary between empty body, string "null", and JSON objects with `reason`. A consumer implementing error handling must account for all three formats.

---

### Negative Scenarios Worth Testing

1. `POST /booking` with `checkin` and `checkout` on the same date - same-day booking - verify whether 0-night stay is accepted
2. `PATCH /booking/:id` sending a field that does not exist in the schema - verify whether unknown fields are rejected or ignored
3. `GET /booking?checkin=invalid-date` - verify behavior on malformed date in filter
4. `DELETE /booking/:id` using an expired token - verify 403 vs. another status
5. `PUT /booking/:id` without `Content-Type: application/json` header - verify behavior
6. `GET /booking?firstname=` (empty value) - verify whether empty filter param returns all bookings or none
