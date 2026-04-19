# Section 5 - Defect Reports

---

## SauceDemo

---

### DEF-001 - Cart data persists in localStorage after logout

**Environment:** SauceDemo - https://www.saucedemo.com - Tested on Chrome 124

**Steps to reproduce:**
1. Login as `standard_user / secret_sauce`
2. Add any two products to the cart (cart badge should show "2")
3. Open the burger menu and click "Logout"
4. Confirm you are returned to the login screen
5. Login as `problem_user / secret_sauce`
6. Navigate to the cart via the cart icon

**Expected result:**
The cart is empty. Each user session should be isolated. Logging out should clear all session data including cart contents.

**Actual result:**
The cart contains the two items added by `standard_user`. The cart badge shows "2". The previous user's cart is fully accessible under the new user session.

**Severity:** High

**Business impact:**
On any shared device - a kiosk, shared household computer, or work machine - a subsequent user sees the previous user's shopping selections. In a real e-commerce application this would expose browsing and purchasing intent to other users. For applications with personal or sensitive product categories (healthcare, adult products) this is a privacy issue, not just a UX inconvenience.

---

### DEF-002 - "Reset App State" executes immediately with no confirmation or user feedback

**Environment:** SauceDemo - tested on Chrome 124 and Firefox 125

**Steps to reproduce:**
1. Login as `standard_user`
2. Add three items to the cart
3. Open the burger menu (top left)
4. Click "Reset App State"
5. Observe what happens to the cart

**Expected result:**
Either a confirmation dialog appears before the reset executes, or a visible success notification confirms the action completed. The user should be informed that their cart was cleared.

**Actual result:**
The cart is cleared immediately. No confirmation is requested. No visual feedback is given. The burger menu closes. The cart badge disappears silently. If the user clicked "Reset App State" by mistake (it sits below "Logout" and above "Close Menu"), they have no way to recover their cart.

**Severity:** Medium

**Business impact:**
Cart abandonment costs are a measurable business metric in e-commerce. A mechanism that silently destroys cart contents without confirmation or undo is a conversion risk. In a production system, this pattern would likely be flagged by UX review before release.

---

### DEF-003 - Checkout form accepts any string as postal code - no format validation

**Environment:** SauceDemo - tested on Chrome 124

**Steps to reproduce:**
1. Login as `standard_user`
2. Add any item to cart
3. Click the cart icon and proceed to checkout
4. In the "Zip/Postal Code" field, enter a single character: `1`
5. Fill first name and last name with any values
6. Click "Continue"

**Expected result:**
The form rejects the input with a validation error indicating an invalid postal code format.

**Actual result:**
The form accepts "1" as a valid postal code. The user proceeds to the checkout overview (step 2) and can complete the order.

**Severity:** Medium

**Business impact:**
In a real shipping system, an invalid postal code causes delivery failure. The order would be created with an undeliverable address. Discovery would happen downstream - at the warehouse or postal service level - rather than at the point of data entry where correction is easiest and cheapest. In Germany, postal codes are always 5 digits. Accepting "1" would generate data that breaks address validation APIs in any downstream integration.

---

### DEF-004 - Sort selection resets to default after navigating to a product page and returning

**Environment:** SauceDemo - tested on Chrome 124

**Steps to reproduce:**
1. Login as `standard_user`
2. On the inventory page, change the sort dropdown to "Price (low to high)"
3. Verify the products reorder correctly
4. Click on any product name to open the product detail page
5. Click the "Back to products" link
6. Observe the sort dropdown

**Expected result:**
The sort selection is preserved. Products remain sorted by "Price (low to high)".

**Actual result:**
The sort dropdown has reset to "Name (A to Z)". Products are displayed in their default sort order. The user's sort preference was lost on navigation.

**Severity:** Low

**Business impact:**
In a real catalog with many products, this forces users to re-apply their sort preference on every navigation step. The UX impact compounds on longer browsing sessions - particularly for users comparing products by price across multiple detail pages. It would appear in usability testing as a repeated friction point.

---

### DEF-005 - error_user can add items to cart but checkout button is non-functional - no error shown

**Environment:** SauceDemo - https://www.saucedemo.com - Chrome 124

**Steps to reproduce:**
1. Login with `error_user / secret_sauce`
2. Add any item to the cart from the inventory page
3. Click the cart icon to navigate to the cart page
4. Click the "Checkout" button

**Expected result:**
The user is taken to checkout step 1 (personal details entry) - the same flow available to `standard_user`.

**Actual result:**
The Checkout button does not navigate anywhere. The page remains on the cart. No error message is displayed to the user. The browser console shows a JavaScript error, but this is not surfaced in the UI.

**Severity:** High

**Business impact:**
A user who is affected by this class of runtime error has no indication that checkout is unavailable. They have added items to their cart and clicked the checkout button - a clear intent to purchase. Receiving no response and no explanation results in a lost conversion with no opportunity for recovery. In a real e-commerce system, silent failures at the checkout entry point are among the most costly possible defects.

---

### DEF-006 - Checkout confirmation provides no order reference number or purchase summary

**Environment:** SauceDemo - https://www.saucedemo.com - All browsers

**Steps to reproduce:**
1. Login as `standard_user / secret_sauce`
2. Add any items to cart
3. Complete checkout steps 1 and 2 with any valid-looking inputs
4. Click "Finish" on the checkout overview
5. Observe the confirmation screen

**Expected result:**
The confirmation screen displays a unique order reference number, a summary of items purchased, and guidance on next steps (e.g. confirmation email, delivery timeframe).

**Actual result:**
The screen shows the message "Thank you for your order!" and a "Back Home" button. No order ID, no item summary, no reference the user can use to follow up. Navigating back from this screen shows no order history anywhere in the application.

**Severity:** Medium

**Business impact:**
Without an order reference, customers have no artifact to reference in a support query. Support teams cannot look up orders because no record exists. In e-commerce operations, the inability to reference a specific transaction is a fundamental gap - it affects customer service handling, dispute resolution, and fraud investigation. The absence of a confirmation email or reference number also increases "did my order go through?" contacts to support, which is a measurable operational cost.

---

## ReqRes

---

### DEF-007 - Page 0 and negative page values return first page data without error

**Environment:** ReqRes API - https://reqres.in

**Steps to reproduce:**
1. Send `GET https://reqres.in/api/users?page=0` with a valid `x-api-key` header
2. Observe the response status and `page` field in the body
3. Send `GET https://reqres.in/api/users?page=-1`
4. Compare responses

**Expected result:**
Status 400 or 422 with an error indicating the page parameter is invalid. Page 0 and negative values are not valid pagination inputs.

**Actual result:**
Status 200 is returned with user data from page 1. The `page` field in the response body reflects the invalid value (e.g. `"page": 0`) even though the actual data returned is from page 1. There is no error indicator.

**Severity:** Low

**Business impact:**
Pagination logic in consumer applications often relies on the `page` field in the response to track current position. If a consumer sends `page=0` and receives `page: 0` with valid data, they may implement broken pagination loops that fail silently - specifically, infinite loops that keep requesting page 0 and receiving the same first page without advancing.

---

### DEF-008 - DELETE on non-existent user ID returns 204 - identical to successful delete

**Environment:** ReqRes API - https://reqres.in

**Steps to reproduce:**
1. Send `DELETE https://reqres.in/api/users/99999` with a valid `x-api-key` header
2. Observe the response status

**Expected result:**
Status 404 - the resource does not exist and cannot be deleted.

**Actual result:**
Status 204 - the same response returned when a real resource is deleted. The response is indistinguishable from a successful delete.

**Severity:** Medium

**Business impact:**
Any consumer using this API pattern learns that DELETE is idempotent and always succeeds. When they integrate with a real backend, their error handling for failed deletes will not exist. Specifically: UI confirmation dialogs like "item deleted" will fire even when nothing was deleted, and retry logic will not be implemented because the consumer has no reason to expect a delete can fail.

---

### DEF-009 - POST /api/users accepts empty body and whitespace-only values with 201

**Environment:** ReqRes API - https://reqres.in

**Steps to reproduce:**
1. Send `POST https://reqres.in/api/users` with body `{}` and valid `x-api-key` header
2. Observe the response status and body
3. Repeat with body `{ "name": "   ", "job": "   " }` (whitespace only)

**Expected result:**
Status 400 or 422 with a validation error. Creating a user without a name or with whitespace-only fields should be rejected.

**Actual result:**
Status 201. For empty body: response contains only `id` and `createdAt`. For whitespace-only values: response contains `name: "   "` and `job: "   "`.

**Severity:** Low

**Business impact:**
Applications consuming this API do not need to implement client-side validation if they assume the server will reject bad input. When they move to a production backend with real validation, empty string handling that passed in development will start failing in production. The API teaches incorrect assumptions about server-side validation standards.

---

## Restful Booker

---

### DEF-010 - POST /auth returns HTTP 200 for authentication failures

**Environment:** Restful Booker API - https://restful-booker.herokuapp.com

**Steps to reproduce:**
1. Send `POST https://restful-booker.herokuapp.com/auth` with body `{ "username": "admin", "password": "wrongpassword" }`
2. Observe the HTTP status code and response body

**Expected result:**
HTTP 401 Unauthorized. The response body may contain an error message.

**Actual result:**
HTTP 200 OK. Response body: `{ "reason": "Bad credentials" }`. The status code indicates success while the body indicates failure.

**Severity:** High

**Business impact:**
Any consumer checking only the HTTP status code (standard REST practice) will treat an authentication failure as success. The consumer must parse the body and check for the `reason` field to detect failure. This is non-standard behavior that breaks every HTTP client that follows REST conventions. Automated auth flows that check for 2xx status before storing a token will store nothing (or an undefined value) and proceed without error - causing downstream 403 failures that are difficult to trace back to the auth step.

---

### DEF-011 - Booking creation accepts checkout date earlier than checkin date

**Environment:** Restful Booker API - https://restful-booker.herokuapp.com

**Steps to reproduce:**
1. Obtain a valid auth token via `POST /auth`
2. Send `POST /booking` with body:
   ```json
   {
     "firstname": "Test",
     "lastname": "User",
     "totalprice": 150,
     "depositpaid": true,
     "bookingdates": {
       "checkin": "2025-06-30",
       "checkout": "2025-06-01"
     },
     "additionalneeds": "None"
   }
   ```
3. Observe the response status and the created booking

**Expected result:**
HTTP 400 with a validation error. A checkout date that precedes the checkin date is logically invalid and should be rejected at the API level.

**Actual result:**
HTTP 200. The booking is created successfully with the invalid date pair. The booking is retrievable and shows `checkin: "2025-06-30"` and `checkout: "2025-06-01"`.

**Severity:** High

**Business impact:**
In a real hotel booking system, an accepted booking with inverted dates would corrupt room availability calculations, produce negative stay-duration values in reporting, generate incorrect pricing (price per night multiplied by a negative number), and appear in calendar systems in undefined states. This is a data integrity failure that would propagate through every downstream system that consumes booking data.

---

### DEF-012 - Non-numeric totalprice is silently coerced to null

**Environment:** Restful Booker API - https://restful-booker.herokuapp.com

**Steps to reproduce:**
1. Send `POST /booking` with `"totalprice": "not-a-number"` (string value)
2. Observe the response status
3. Retrieve the created booking with `GET /booking/:id`
4. Check the `totalprice` field in the GET response

**Expected result:**
HTTP 400 with a validation error. A non-numeric value for a price field should be rejected.

**Actual result:**
`POST /booking` returns HTTP 200. The booking is created. `GET /booking/:id` returns the booking with `"totalprice": null`.

No error is returned at any point. The invalid input is accepted, transformed to null, and stored silently.

**Severity:** Medium

**Business impact:**
Downstream revenue reporting, invoicing, and price-based filtering queries will encounter null values where a price is expected. The booking is in the system but cannot be included in any financial calculation. The consumer who created the booking receives no indication that their submitted price was invalid - they assume the booking was created correctly. Discovery happens later, in reporting or invoicing, where the root cause (bad input at creation time) is difficult to trace.

---

### DEF-013 - DELETE /booking/:id returns 201 Created instead of 204 No Content

**Environment:** Restful Booker API - https://restful-booker.herokuapp.com

**Steps to reproduce:**
1. Create a booking and note the `bookingid`
2. Send `DELETE /booking/:id` with a valid `Cookie: token=<value>` header
3. Observe the HTTP status code

**Expected result:**
HTTP 204 No Content. This is the correct HTTP semantics for a successful delete with no response body.

**Actual result:**
HTTP 201 Created. The resource was deleted, but the status code indicates a resource was created. There is no response body.

**Severity:** Medium

**Business impact:**
HTTP clients and integration frameworks that follow REST standards treat 201 as a creation event and 204 as a successful delete. A 201 on a DELETE request will confuse any framework that routes responses by status code. Specifically: webhook handlers, event-driven consumers, and audit logging systems that derive action type from HTTP method + status code will misclassify the event as a creation. This will cause incorrect audit trails and false-positive alerts in monitoring systems looking for unexpected resource creation events.
