# Section 3 - High-Value Test Design

Scenarios are written as intent + expected outcome. Not step-by-step scripts.

---

## SauceDemo

### Login

**SC-01 - Standard login with valid credentials**
Login with `standard_user / secret_sauce`. Expect redirect to `/inventory.html` and the product list visible with all items.

**SC-02 - Login with locked account**
Login with `locked_out_user / secret_sauce`. Expect an error message on the login page and no redirect. Verify the message is meaningful and not a generic error code.

**SC-03 - Login with empty fields**
Submit login form with both fields empty. Expect inline validation error, no page navigation. Verify error references both fields.

**SC-04 - Login with correct username, wrong password**
Submit with `standard_user` and an incorrect password. Expect an error. Verify the error does not reveal which field is wrong (username vs. password).

---

### Product Catalog and Sort

**SC-05 - Sort by price low to high**
From the inventory page, select "Price (low to high)". Verify the displayed prices are in ascending order across all visible items. Note whether the sort applies to the current page only or all products.

**SC-06 - Sort by name Z to A**
Select "Name (Z to A)". Verify product names are in reverse alphabetical order. Then navigate to a product detail page and return. Verify sort selection has been preserved or has reset - either behavior is acceptable but it must be consistent.

**SC-07 - Sort behavior with problem_user**
Login as `problem_user`. Attempt to sort by price. Observe and document the actual behavior - broken sort is expected and should be confirmed reproducible.

---

### Cart and Session

**SC-08 - Cart persists across logout and re-login**
Add items to cart as `standard_user`. Logout. Log back in as the same user. Verify cart contents are present. Then logout and login as a different valid user. Verify whether the other user's session inherits the cart. This is the critical scenario.

**SC-09 - Add and remove items from cart**
Add three items. Navigate to cart. Remove one. Return to inventory and verify cart badge shows 2. Add one item again. Verify badge updates to 3.

**SC-10 - "Reset App State" effect on cart**
Add items to cart. Open burger menu and click "Reset App State". Verify cart is cleared. Verify there is no confirmation before execution. Document whether the user receives any visual feedback.

---

### Checkout

**SC-11 - Complete checkout with valid inputs**
Add at least two items. Proceed through all checkout steps with valid first name, last name, and postal code. Verify the overview step shows correct item subtotal, correct 8% tax, and correct total. Complete checkout and verify a success screen appears.

**SC-12 - Checkout with single-character postal code**
At checkout step 1, enter a single character (e.g. "1") in the postal code field. Attempt to proceed. If the form accepts it, document this as a validation gap. Track whether the checkout completes successfully with this input.

**SC-13 - Checkout back-navigation behavior**
On checkout step 2 (overview), click "Cancel". Verify the user returns to the product list and the cart still contains all added items. Verify the entered personal details from step 1 are cleared (cannot be retrieved without re-entering).

**SC-14 - Checkout with empty required field**
On step 1, leave the first name field empty and proceed. Expect an inline error message. Verify the message identifies the specific missing field.

---

### Multi-user Behavioral Coverage

**SC-15 - performance_glitch_user checkout timing**
Login as `performance_glitch_user`. Add an item and proceed through checkout. Document the response time at each step. Verify the application completes the flow eventually despite delays - confirm no timeout or failure occurs during normal usage.

---

## ReqRes

### User Endpoints

**SC-16 - GET users with valid page numbers**
Request page 1 and page 2. Verify page 1 returns 6 users, page 2 returns 6 users. Verify `total` is 12 and `total_pages` is 2 across both responses. Verify no user appears on both pages.

**SC-17 - GET users with out-of-range page**
Request page 3. Verify the response returns an empty `data` array (not a 404). Confirm `page` in the response matches the requested page number.

**SC-18 - GET single user: valid and non-existent IDs**
GET `/api/users/1` through `/api/users/12` - all should return 200 with user data. GET `/api/users/13` and higher - verify 404 with an empty body or a consistent error structure.

**SC-19 - POST user with complete payload**
POST with `{ "name": "Test User", "job": "QA Engineer" }`. Verify response is 201. Verify response contains `id`, `name`, `job`, and `createdAt`. Verify `createdAt` is a valid ISO timestamp.

**SC-20 - POST user with empty payload**
POST with `{}`. Document the status code returned. If 201, document the response structure and flag that server-side validation is absent.

**SC-21 - DELETE non-existent user**
DELETE `/api/users/9999`. Verify the status code. If it returns 204 (same as a real delete), document the inconsistency - the API cannot distinguish between deleting a real user and a no-op.

---

## Restful Booker

### Booking CRUD

**SC-22 - Create booking with all fields**
POST a booking with valid `firstname`, `lastname`, `totalprice` (positive integer), `depositpaid` (boolean), valid date pair (checkin before checkout), and `additionalneeds` string. Verify 200, verify returned `bookingid` is an integer, verify returned `booking` object matches submitted data exactly.

**SC-23 - Get booking by ID**
After creating a booking, GET it by the returned ID. Verify the response body matches what was submitted. Pay attention to `totalprice` type (integer vs. float), date format, and `depositpaid` value.

**SC-24 - Create booking with checkout before checkin**
Submit a booking where `checkout` is earlier than `checkin`. Document whether the API accepts or rejects this. If accepted (returns 200), this is a validation gap.

**SC-25 - Create booking with negative totalprice**
Submit `totalprice: -200`. Document the response. If 200, note that the API accepts logically invalid financial data.

**SC-26 - Update booking without auth token**
Attempt PUT or PATCH on a valid booking ID without the Cookie header. Verify 403. Verify the response body does not reveal the booking data.

**SC-27 - Delete booking and verify it is gone**
Delete a booking by ID using a valid token. Verify the response is `201` (known quirk - document this). Then GET the same ID. Verify the response is 404.

**SC-28 - Filter bookings by firstname**
Create a booking with a unique first name. GET `/booking?firstname=<that name>`. Verify the booking ID appears in the results. Create a second booking with a different name and verify it does not appear in filtered results.
