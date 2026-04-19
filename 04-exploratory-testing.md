# Section 4 - Exploratory Testing Sessions

Sessions are structured but not scripted. The goal is to find what structured tests miss.

---

## SauceDemo

---

### Session 1 - Cart state and session boundary behavior

**Charter:** Understand how the application manages cart data across login, logout, and user switches. Determine whether session boundaries are enforced.

**Timebox:** 45 minutes

**Scope:** Login/logout flow, localStorage, cart state across user sessions

---

**Observations:**

I started by adding three items to the cart as `standard_user`, then logged out via the burger menu. At the login screen I inspected localStorage in browser devtools - the cart data was still present. I logged in as `problem_user` and navigated to the cart. The items from the previous session were visible under the new user's session.

This confirmed that logout does not clear localStorage. The application's logout only removes the `session-username` key but leaves `cart-contents` (or equivalent) intact.

I then tested whether "Reset App State" from the burger menu clears localStorage. It does. The cart data was removed. However, there is no confirmation step and no visual toast or success message - the action is immediate and silent. A user who clicks it by mistake loses their cart with no undo.

I tested the cart badge count after a "Reset App State" call. The badge disappeared immediately, which is correct. However, when I added items to the cart again after a reset, the badge updated normally. The reset does not leave the application in a broken state - just a silent one.

I noted that the checkout flow does not write any partially-entered data to localStorage. If a user completes step 1 (personal details) and closes the browser, the cart persists but the entered name and postal code are gone. There is no draft checkout concept.

I attempted login with `error_user`. The login succeeded. Adding items to the cart worked. However, clicking the cart icon produced a JavaScript error (visible in browser console). The cart page rendered but the "Checkout" button did not function - clicking it did nothing. This means `error_user` can add items but cannot complete a purchase.

---

**Issues found:**

- Cart data persists in localStorage after logout. Any user on the same browser inherits the previous user's cart at next login. [DEF-001]
- "Reset App State" is destructive and has no confirmation or feedback. [DEF-002]
- `error_user` can add items to cart but cannot initiate checkout. The failure is silent - no error message is shown to the user.
- Browser console shows JavaScript errors for `error_user` during cart interaction. These are swallowed silently in the UI.

---

**Insights:**

Session boundary failures like the localStorage issue are not visible in happy-path testing. They only appear when you test the transition between users or browser sessions. In a real e-commerce application this would be a privacy and data integrity issue - a shared device (public kiosk, shared family laptop) would expose previous users' cart contents.

The "Reset App State" UX problem is worth raising with product, not just development. The behavior is technically working as designed, but the design itself is risky for real users.

---

### Session 2 - Checkout flow integrity and edge case behavior

**Charter:** Probe the checkout flow for validation gaps, state inconsistencies, and edge case handling beyond the happy path.

**Timebox:** 40 minutes

**Scope:** Checkout steps 1-3, form validation, price calculation, navigation behavior

---

**Observations:**

I started checkout with a cart containing all six available products. On step 1, I tested the fields individually. First name and last name fields accepted numbers, special characters, and single characters without any complaint. The postal code field accepted "1" as valid input. The form proceeded to step 2 with these values.

On step 2 (overview) I verified the price calculations manually. With all six items the item total was $129.94. Tax at 8% should be $10.40. The displayed tax was $10.40 - correct. The total was $140.34 - correct. The calculation is accurate.

I then clicked the browser back button from step 2. The browser navigated back to step 1, but the previously entered name and postal code were blank. The form reset completely. This is an expected browser behavior, but in a user journey context it is friction - the user must re-enter all details.

I tested cancellation from step 2 using the "Cancel" button. This returned me to the product list, not the cart. The cart was intact. The behavior is sensible but counter-intuitive - a cancel from checkout overview should arguably return to the cart, not the full product list.

I completed checkout successfully. The confirmation screen shows "Thank you for your order!" and a "Back Home" button. No order reference, no summary of what was purchased, no email mentioned. After clicking "Back Home" the cart badge was gone and the inventory was displayed normally.

I attempted to navigate directly back to the checkout confirmation URL after completing an order. The URL did not resolve to a specific order - navigating back showed the checkout step 1 form (empty), not the confirmation. There is no persistent order record.

I tested checkout with `performance_glitch_user`. Each step button click had a 2-4 second delay before the page responded. The flow completed without failure. I observed that there is no loading indicator during these delays - the button appears to do nothing for several seconds. A real user would likely click it again, which could result in duplicate submissions in a production system.

---

**Issues found:**

- Checkout form accepts single character and numeric input for all fields including postal code. No format validation. [DEF-003]
- Checkout confirmation provides no order reference, no purchased items summary, and no next steps. Real users have no confirmation artifact.
- Cancel from checkout step 2 returns to product list instead of cart. Unexpected navigation.
- `performance_glitch_user` shows no loading state during delayed responses. Double-click risk in production equivalent.

---

**Insights:**

The absence of an order confirmation reference is a business risk, not a UI polish issue. In any real e-commerce context, support teams need an order ID to handle customer queries. Here there is nothing to reference.

The postal code validation gap matters because in Germany and other European markets, a five-digit postal code is structurally required for address processing. An application that accepts "1" as a postal code would generate undeliverable shipments silently.

---

## ReqRes

### Session 3 - API contract and boundary behavior

**Charter:** Probe the ReqRes API response contracts, boundary handling, and behavior consistency across endpoints.

**Timebox:** 30 minutes

**Scope:** User list pagination, write endpoint validation, HTTP semantics

---

**Observations:**

I started by mapping the pagination behavior. Page 1 and page 2 return 6 users each with consistent `per_page`, `total`, and `total_pages` values. Page 3 returns an empty `data` array with `total: 12` and `total_pages: 2` - the metadata is accurate even when data is empty. This is good API design.

I requested `page=0` and `page=-1`. Both returned identical responses to `page=1` - no validation error, no indication that the parameter was out of range. The `page` field in the response matched the requested value (0 and -1 respectively) even though the data returned was the first page. This is misleading.

I sent a `POST /api/users` with an empty body. Response was 201 with `{"id":"742","createdAt":"2024-..."}`. No `name` or `job` in the response because none were sent. The server accepted an empty record silently.

I then sent POST with `{ "name": " ", "job": " " }` (whitespace only). Also 201. Whitespace is treated as valid input.

DELETE on ID 9999 returned 204. No distinction between a real delete and a non-existent resource. I deleted the same non-existent ID three times in a row - all 204. The API has no state to distinguish these.

PUT on a seeded user (ID 1) returned 200 with `updatedAt`. I then GET `/api/users/1` - the user data was unchanged. The PUT appeared to succeed but made no change. This is expected for a stateless mock but the 200 response implies success that did not occur.

---

**Issues found:**

- `page=0` and `page=-1` return data instead of a 400 or 422 validation error. [DEF-007]
- DELETE on non-existent IDs returns 204 - indistinguishable from a real delete. [DEF-008]
- POST accepts empty body and whitespace-only field values with 201. [DEF-009]
- PUT implies state change (200 + updatedAt) but no change persists.

---

**Insights:**

For a mock API, the stateless behavior is by design. The real risk is that developers building against this API learn incorrect HTTP patterns - specifically that DELETE always succeeds and PUT always writes. When they move to a real backend these assumptions break.

The pagination boundary issue is worth noting. An API that returns page 1 data when page=0 is requested creates confusion in pagination logic. Consumers may believe they are handling an edge case correctly when they are not.

---

## Restful Booker

### Session 4 - Auth boundary and booking data integrity

**Charter:** Test the auth boundary, token lifecycle, and data validation on booking creation and modification.

**Timebox:** 45 minutes

**Scope:** Auth endpoint, token usage, booking field validation, update operations

---

**Observations:**

I started by obtaining a valid token. I then made a series of write requests using that token and monitored when it stopped working. After approximately 10 minutes the token returned 403. This confirms token expiry but the exact TTL was not consistent across sessions - it varied between 8 and 12 minutes. No token refresh mechanism exists. Consumers must re-authenticate.

I tested auth failure behavior. `POST /auth` with wrong password returns `{ "reason": "Bad credentials" }` with status 200. Status 200 for an authentication failure is incorrect HTTP semantics - it should be 401. A consumer checking only the status code would not detect the failure.

I created a booking with `checkin: "2024-12-31"` and `checkout: "2024-01-01"` (checkout before checkin). The API returned 200. The booking was created and retrievable. No validation error.

I created a booking with `totalprice: -999`. Returned 200. Booking was created with `totalprice: -999`. This is logically invalid - a negative room price.

I created a booking with `totalprice: "not-a-number"`. Returned 200. Retrieved the booking and found `totalprice: null`. The coercion happened silently with no indication in the create response.

I sent a `PATCH` request with an empty body `{}`. Response was 200. The booking was unchanged. No error was returned - the server treated a no-op update as success.

I then tested the access control boundary. I created two bookings with the same token. Using that token I deleted a booking that I had not created (I used an ID that existed before my test session). The delete returned 201 and the booking was gone. There is no ownership model - any token can modify or delete any booking.

I tested GET endpoints without authentication. All booking data is publicly accessible - IDs, names, dates, prices. No auth is required to read any booking.

---

**Issues found:**

- `POST /auth` returns 200 for authentication failures. Should return 401. [DEF-010]
- Booking creation accepts checkout date earlier than checkin date. [DEF-011]
- Negative `totalprice` accepted without error. [DEF-012]
- Non-numeric `totalprice` coerced silently to null - no error in response. [DEF-013]
- Any authenticated user can delete bookings they did not create.
- All booking data publicly readable without authentication.

---

**Insights:**

The most serious finding from this session is the access control gap. In a real booking system, allowing any authenticated user to delete any other user's booking would be a critical defect. The auth system verifies identity but does not enforce ownership.

The `POST /auth` returning 200 for failures is a pattern that breaks every consumer who checks status codes. It requires consumers to parse the body to detect failure - non-standard and error-prone.

The date validation gap is business-critical. A checkout-before-checkin booking would corrupt revenue calculations, room availability calendars, and guest management systems downstream.
