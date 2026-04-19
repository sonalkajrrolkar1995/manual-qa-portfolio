# Section 2 - Risk Analysis

---

## SauceDemo

### Functional Risks

**Cart state not cleared on logout.**
Cart contents are stored in localStorage and are not removed when the user logs out. If a second user logs in on the same browser, they inherit the previous session's cart. This is not visible at login - it appears only when navigating to the cart.

**Checkout form accepts any input for all fields.**
First name, last name, and postal code fields have no format validation. A single character is accepted as a valid postal code. This would cause downstream failures in a real system - delivery, invoicing, address parsing.

**Sort selection is lost on navigation.**
If the user changes the sort order, navigates to a product page, and returns to the list, the sort resets to the default. The selected sort state is not persisted in the URL or session.

**problem_user exposes class of real data-driven bugs.**
This user account simulates what happens when product data is malformed - wrong images mapped to products, sort function broken, some interactions failing silently. This class of bug (data-driven rendering failure) is common in production catalog systems.

---

### Data Risks

**No quantity field in cart.**
The current design allows only one unit per product. If this were a real system, there is no mechanism to represent multiple units of the same item.

**No order reference generated.**
The checkout confirmation screen shows no order ID, no reference number, and no summary email. A user completing checkout has no confirmation they can reference.

**Cart data persists indefinitely.**
There is no expiry on localStorage data. A cart created today will still be present in a week on the same browser.

---

### Business Risks

**Locked out user receives an error but no recovery path.**
`locked_out_user` sees an error message but there is no "contact support" link, no account recovery option, and no explanation beyond "this user is locked out." In a real product this is a lost customer.

**Checkout flow has no save/resume capability.**
If the user closes the browser mid-checkout, the cart persists but the entered checkout details (step 1) are lost. There is no draft order concept.

**No confirmation email or order tracking.**
After checkout success there is no actionable confirmation for the customer. Businesses that process real orders require this for customer service and dispute resolution.

---

### Security Risks

**localStorage cart visible to any script on the page.**
Cart contents - which could include product IDs, quantities, and pricing - are readable by any JavaScript running on the same origin. In a real application with third-party scripts this is a data exposure risk.

**Session does not expire.**
No observable session timeout exists. The application remains logged in indefinitely on an inactive browser.

**No CSRF protection observable.**
The checkout form submission has no observable CSRF token. In a real application this would be a high severity finding.

---

### UX Risks

**"Reset App State" executes immediately without confirmation.**
The burger menu option clears the cart and resets the application with no confirmation dialog. This is destructive and silent. A user who clicks it accidentally loses their cart.

**"About" link navigates away in the same tab.**
The About link in the hamburger menu opens saucelabs.com in the current tab, navigating the user away from the application with no back-navigation context preserved.

**No empty cart state guidance.**
When the cart is empty and the user visits the cart page, there is no "Continue shopping" call-to-action. The user must use browser back or navigate manually.

---

## ReqRes

### Functional Risks

**Stateless creates break chained test flows.**
Any integration or test that calls `POST /api/users` and then tries to `GET` the returned ID will fail. The API returns plausible-looking IDs (e.g. 745) that do not exist in the system. This misleads consumers who assume persistence.

**DELETE accepts any ID and returns 204.**
`DELETE /api/users/999999` returns 204 - the same response as a successful delete of a real user. There is no 404 for non-existent resource deletion. Consumers relying on 204 as confirmation of deletion cannot distinguish between a successful delete and a no-op on a non-existent ID.

**Page parameter boundary behavior is unvalidated.**
`GET /api/users?page=0` and `GET /api/users?page=-1` return data rather than a validation error. The behavior on invalid page values is undefined from a contract perspective.

---

### Data Risks

**No input validation on write endpoints.**
`POST /api/users` with `{}` or `{ "name": "", "job": "" }` returns 201. Empty strings, whitespace-only values, and missing fields are all accepted without error.

**Response fields are inconsistent across endpoints.**
`POST` returns `id` and `createdAt`. `PUT` and `PATCH` return `updatedAt` but not the full object. `GET` returns a different field set. There is no consistent envelope structure.

**total_pages is hardcoded and ignores filters.**
`total_pages: 2` is returned regardless of what query parameters are applied. This makes it unreliable for pagination logic.

---

### Business Risks

**The API cannot support any real stateful workflow.**
Because nothing persists, the API is unsuitable for testing any feature that requires reading back what was created. This is a significant limitation for teams trying to use it as a staging environment mock.

---

### Security Risks

**x-api-key is validated at entry but not scoped.**
Any valid key has access to all endpoints and all operations. There is no role-based access, no per-key rate limiting that can be observed, and no differentiation between read-only and write-capable keys.

**No observable rate limiting.**
Rapid sequential requests receive normal responses. This makes the API unsuitable for any security resilience testing.

---

## Restful Booker

### Functional Risks

**No date order validation.**
A booking can be created with a `checkin` date after the `checkout` date. The API returns 200. This is a logic error - a guest cannot arrive after they leave.

**No price validation.**
`totalprice` accepts negative integers and returns 200. A booking with `totalprice: -500` is created successfully.

**totalprice coercion hides errors.**
Non-numeric strings are coerced to `null` and the booking is created. The consumer receives no error - they receive a 200 with `totalprice: null`. This is a silent data corruption.

**PATCH with empty body returns 200.**
Sending a PATCH request with `{}` returns 200. Nothing changes, but the server reports success. This makes it impossible for consumers to detect a failed partial update.

**Token expiry is not documented.**
Auth tokens expire (approximately 10 minutes based on observation) but the expiry time is not documented and the API does not return an `expires_at` field. Consumers have no way to proactively refresh tokens.

---

### Data Risks

**Booking IDs are sequential integers.**
IDs are auto-incremented. This means any authenticated user can enumerate all bookings by iterating from ID 1 upwards. There is no UUID or non-predictable ID scheme.

**No pagination on GET /booking.**
`GET /booking` returns all booking IDs in a single response. At scale this becomes a performance and memory issue for both the server and the consumer.

**additionalneeds field is unstructured.**
This is a free-text field with no length limit or validation. It can contain anything.

---

### Business Risks

**Any authenticated user can delete any booking.**
There is no ownership check. A token obtained with valid credentials can delete bookings created by any other user. In a real booking system this would be a critical access control failure.

**DELETE returns 201 instead of 204.**
This breaks standard REST conventions and will cause integration failures with any client that validates the status code of a delete operation.

**No booking conflict detection.**
Two bookings for the same guest (same name, same dates) can be created without any uniqueness check or conflict warning.

---

### Security Risks

**All bookings are publicly readable without authentication.**
`GET /booking` and `GET /booking/:id` require no token. Guest names, dates, and pricing are visible to unauthenticated requests.

**Auth credentials are hardcoded and publicly known.**
`admin / password123` is documented publicly. In a real system this would mean any person who has read the documentation can authenticate.

**Old tokens remain valid after re-authentication.**
Generating a new token does not invalidate the old one. Token revocation is not supported.
