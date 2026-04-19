# Section 7 - System Thinking

Senior-level risk analysis. Not about individual features - about how systems fail at scale, under real conditions, and at integration boundaries.

---

## SauceDemo

### System-Level Failure Risks

**Client-side state as single source of truth.**
The entire application state lives in localStorage. There is no server, no database, no session store. This architecture means the application cannot recover from localStorage corruption, browser storage limits, or privacy mode sessions. In private/incognito mode, localStorage is cleared when the tab closes - the cart is gone. A user mid-checkout who opens a second private tab starts with an empty cart.

More critically: localStorage is origin-scoped but not user-scoped. Any script served from the same origin can read and modify cart state. In a real application with third-party analytics or tag manager scripts, this is a data leakage surface.

**No state synchronization between tabs.**
If a user has two tabs open, adding an item in one tab is not reflected in the other until the page is refreshed. Cart badge counts and cart contents can desync. In a real multi-tab shopping session (comparing products across tabs), this would produce incorrect cart state.

**Test user matrix is not systematically covered in most test strategies.**
The six user accounts are not cosmetic - each represents a real failure class. `performance_glitch_user` simulates slow infrastructure. `problem_user` simulates data corruption. `visual_user` simulates rendering failures. `error_user` simulates JavaScript runtime errors. A test strategy that covers only `standard_user` and `locked_out_user` leaves four classes of production failure untested.

---

### Data Flow Issues

The checkout flow collects name, postal code, and nothing else. No email, no shipping address, no payment data. The data that does get collected has no validation and goes nowhere. In a real system this is the integration point - the collected data would go to an order management system, a payment processor, and a shipping provider. Each of those systems would have their own validation. Any mismatch between what the checkout form accepts and what downstream systems require would produce failures that are invisible at the UI level.

The order confirmation screen generates no ID and no record. There is no data flow after checkout. In a real system the absence of an order confirmation ID means customer support has no record to look up and the customer has no reference for a dispute.

---

### Integration Risks

SauceDemo has no integrations - it is a self-contained frontend. But the patterns it establishes are worth noting for integration risk planning:

- If this frontend were connected to a real backend, the missing input validation on checkout fields would mean invalid data reaching the backend. Backend validation would need to compensate.
- The localStorage cart model is incompatible with server-side cart management. A migration to server-side carts would require a synchronization strategy and a way to handle the conflict between browser state and server state.
- No CSRF tokens on form submissions. A real backend integration would require this to be added.

---

### Reliability Risks

**Single user type tested = single failure mode understood.**
Most teams testing SauceDemo will test `standard_user` and call it done. The reliability risk is that the application behaves differently for five other user types - and these user types represent real production conditions, not synthetic test scenarios.

**No error boundaries visible in the UI.**
When JavaScript errors occur (`error_user`), the application does not display an error page or a fallback. It silently fails. A production application should surface errors to users in a controlled way rather than presenting a non-functioning UI with no explanation.

---

## ReqRes

### System-Level Failure Risks

**Statelessness is not communicated in the API contract.**
The API returns 201 with an ID, which implies the resource was created and can be retrieved. It cannot. The gap between implied contract (persistent REST API) and actual behavior (stateless mock) is not surfaced in the response. Consumers who build integration logic based on this API will discover the statefulness assumption at the worst possible time - in production, not in testing.

**Auth token is decorative.**
The token returned by `POST /api/login` is a static value that is not checked by any other endpoint. It cannot be used to authorize requests because no endpoint requires it (the x-api-key is separate). A consumer who builds a login flow, stores the token, and attaches it to subsequent requests will not know the token is doing nothing. This creates a false sense of auth security.

---

### Data Flow Issues

There is no data flow. Creates, updates, and deletes all return successful responses but produce no state change. The seeded data is permanent.

The risk this creates is not in ReqRes itself but in what consumers learn. Developers who practice against this API learn that: POST always returns 201, DELETE always returns 204, and the resource ID in the POST response can be used in subsequent operations. All three of these assumptions are false in production systems with real validation, real state, and real IDs.

---

### Integration Risks

**The `support` object in every response.**
Every list endpoint includes `{ "support": { "url": "...", "text": "..." } }`. A consumer that assumes a response object has only `data` and pagination fields will encounter an unexpected key. If the consumer is strict about unknown fields (many production deserializers are), this will cause a parse failure.

**Inconsistent error structure.**
404 returns `{}`. 400 returns `{ "error": "..." }`. 201 returns a flat object. 200 (list) returns `{ "data": [...], "support": {...} }`. Any consumer that writes a generic response handler must account for all four patterns. This inconsistency is a real integration overhead.

---

### Reliability Risks

The API is hosted externally. It has been unavailable in the past (rate limiting, maintenance). Any test suite or development workflow that depends on it will fail during outages. This is a team-level risk: teams using ReqRes for CI validation will see intermittent failures that are not related to their code.

---

## Restful Booker

### System-Level Failure Risks

**Auth model does not enforce resource ownership.**
The API has authentication but no authorization. A valid token proves identity but not permission. Any authenticated user can read, update, or delete any other user's booking. At system level this means the auth layer is incomplete. Authentication without authorization is not a security model - it is an identification model. Any system that assumes "authenticated = authorized to act on any resource" will have data integrity and privacy failures at scale.

**Token expiry without a refresh mechanism.**
Tokens expire in approximately 10 minutes. There is no `refresh_token`, no `expires_at` in the auth response, and no way to extend a token. Consumers must re-authenticate. This is fine for short-lived operations but any long-running process (a booking import script, a nightly report, a webhook consumer) will have its token expire mid-operation. Without explicit handling, this results in silent 403s mid-process with no recovery.

**DELETE returns 201 - a design-level HTTP semantics failure.**
This is not a surface-level bug. It indicates the endpoint was implemented without validation against HTTP conventions. If this pattern exists in one endpoint, it suggests other endpoints may also have undocumented deviations from HTTP standards. Consumers who follow the spec will build integration code that fails on this endpoint specifically.

---

### Data Flow Issues

**No booking conflict detection at the API level.**
Two bookings with identical guest names and overlapping dates can be created without any uniqueness check. In a real booking system, the API is the last defense before conflicting bookings reach the availability calendar, revenue management system, and housekeeping schedule. An API that accepts duplicate bookings silently creates downstream reconciliation work in every connected system.

**Silent type coercion on totalprice.**
A non-numeric `totalprice` is coerced to `null` and stored. The booking appears valid to any downstream system that does not check for null prices. Revenue calculations, tax computation, and payment processing will encounter null where a number is expected. The failure point is downstream - not at the API where the bad data entered the system.

**Date validation is missing at the one place it matters most.**
The API is the single entry point for booking data. If invalid dates enter here, every downstream system (calendar, reporting, billing) receives invalid data. There is no second validation layer downstream because all downstream systems assume the API enforces data integrity. The API does not.

---

### Integration Risks

**No pagination on GET /booking.**
As booking volume grows, `GET /booking` returns the full dataset in a single response. At 10,000 bookings this becomes a slow, large payload. At 100,000 it becomes a practical failure. Any consumer that calls `GET /booking` to display a list will begin to see timeouts as data grows. Adding pagination later is a breaking change to the API contract.

**Inconsistent 404 response body.**
`GET /booking/:id` on a non-existent ID returns the string `"null"` as the body, not valid JSON. Any consumer using a JSON parser on the 404 response will throw a parse error. This is an integration failure point that appears only on the not-found code path - it will not appear in happy-path testing and will only surface when a consumer first encounters a deleted or non-existent booking in production.

**auth header pattern is unusual.**
The Cookie-based auth (`Cookie: token=xxx`) is not standard token auth. Most HTTP clients and frameworks have built-in support for `Authorization: Bearer <token>`. The Cookie pattern requires custom header injection. Teams building integrations will likely attempt Bearer auth first, see it fail, and spend time diagnosing an underdocumented auth pattern.

---

### Cross-System Reliability Observations

All three systems are externally hosted with no SLA. ReqRes and Restful Booker are public free services. SauceDemo is Sauce Labs-maintained. Test suites dependent on any of these will experience intermittent failures during outages or rate limit events that are not caused by the code under test.

For a production team using these systems as mock backends in CI:
- Intermittent failures corrupt CI reliability metrics
- Teams begin to distrust failing CI runs ("probably just ReqRes being flaky again")
- Real failures get masked by expected flakiness

This is a team and process risk, not a code risk. The mitigation is to treat these external systems as integration tests that can be skipped on infrastructure failure, not as unit-level tests that must always pass.
