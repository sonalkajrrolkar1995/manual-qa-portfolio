# Section 1 - Product Understanding

---

## SauceDemo

SauceDemo is a demo e-commerce application by Sauce Labs. It simulates a webshop: login, product catalog, shopping cart, and a three-step checkout flow. There is no real backend. All state - cart contents, session - is kept in browser localStorage.

**Critical business flows:**

1. Login -> product list -> add to cart -> checkout -> order confirmation
2. Login -> sort/browse products -> modify cart -> abandon checkout
3. Login -> complete checkout in one uninterrupted session

Six hardcoded user accounts exist, each producing different application behavior. `standard_user` is the working baseline. `locked_out_user` is blocked at login. `problem_user` has broken product images and a non-functional sort dropdown. `performance_glitch_user` adds 2-5 second delays to every action. `error_user` triggers JavaScript errors on specific interactions. `visual_user` shows layout and rendering defects.

Any test strategy covering only `standard_user` misses a large surface area. These users simulate real-world conditions: degraded data, slow infrastructure, rendering bugs. They are not just test fixtures.

Checkout collects first name, last name, and postal code. Tax is calculated at 8% of item total. No payment processing occurs and no order reference number is generated on the confirmation screen.

---

## ReqRes

ReqRes is a hosted mock REST API simulating user management. It covers listing, retrieving, creating, updating, and deleting users, plus basic authentication endpoints.

**Critical flows:**

1. Authenticate with email and password -> receive token
2. List users with pagination -> retrieve single user by ID
3. Create user -> receive ID and createdAt timestamp in response
4. Update user full (PUT) or partial (PATCH) -> receive updatedAt timestamp
5. Delete user -> receive 204

The API is stateless. `POST /api/users` returns an ID and timestamp, but nothing persists. A subsequent `GET` using that returned ID will 404 unless it falls within the 1-12 range of seeded users. Any test chaining create then read on the new ID will fail. This is not a bug - it is how the API is designed.

The dataset is 12 seeded users across 2 pages (6 per page). These are static and always return the same data regardless of any write operations.

The `x-api-key` header is required on all requests since late 2024. Without it, all requests return 401.

---

## Restful Booker

Restful Booker is a REST API simulating a hotel booking system. It supports full CRUD on bookings with token-based authentication for write operations.

**Critical flows:**

1. Get auth token -> create booking -> read it -> update it -> delete it
2. List all bookings -> filter by guest name or date -> retrieve specific booking
3. Attempt write operation without token -> expect 403
4. Attempt write with expired or invalid token -> expect 403

Auth token is obtained from `POST /auth` with credentials `admin / password123`. On failure the response is `{ "reason": "Bad credentials" }` - there is no token field in the failure response.

A booking object has six fields: `firstname`, `lastname`, `totalprice`, `depositpaid`, `bookingdates` (with `checkin` and `checkout`), and `additionalneeds`. The API's field-level validation is inconsistent and undocumented.

Write operations require `Cookie: token=<value>`. HTTP Basic auth is also accepted as an alternative. `DELETE /booking/:id` returns `201 Created` instead of `204 No Content` - this is a persistent defect in the application, not a documentation issue.
