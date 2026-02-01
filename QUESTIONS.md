## 1. If this app had 10,000 employees checking in simultaneously, what would break first? How would you fix it?

the **database layer would break first**, not the API code.

Right now, the application uses **SQLite**. SQLite allows only one write operation at a time. If thousands of employees try to check in together, write requests would start queuing up, response times would increase, and the Node.js server could feel blocked because database calls are effectively synchronous.

### How I would fix it:

**Short-term :**
- Keep the check-in API lightweight and fast.
- Make sure proper indexes exist on frequently queried columns.
- Fail fast if an employee already has an active check-in.

**Medium-term :**
- Migrate from SQLite to **PostgreSQL or MySQL**.
- Use connection pooling so multiple writes can happen concurrently.

**Long-term :**
- Introduce a **queue-based architecture** (Redis / Kafka).
- API servers accept requests and push them to a queue.
- Background workers handle database writes.
- Scale API servers horizontally behind a load balancer.

the **database choice and write strategy** need to evolve for high concurrency.

---

## 2. The current JWT implementation has a security issue. What is it and how would you improve it?

The main issue is that the JWT access token is stored in **localStorage** on the frontend.

Storing tokens in localStorage makes them vulnerable to **XSS attacks**. If any malicious script runs in the browser, it can read and steal the token.

### How I would improve it:

- Use **short-lived access tokens** (10–15 minutes).
- Store **refresh tokens in HttpOnly cookies**, which JavaScript cannot access.
- Implement a `/refresh` endpoint to issue new access tokens when they expire.
- Rotate refresh tokens and allow server-side revocation on logout.

This approach improves security without making the system overly complex.

---

## 3. How would you implement offline check-in support?

Offline support is mostly a **frontend problem**, with a small backend assist.

### My approach:

- Detect offline mode using `navigator.onLine`.
- When offline, save check-ins locally using **IndexedDB** (via a library like `localForage`).
- Each offline check-in gets a client-generated unique ID.
- Show clear UI feedback: *“Check-in saved. Will sync when online.”*

### Syncing later:

- When the device comes back online, send all pending check-ins to a `/checkin/sync` API.
- Backend validates and stores them.
- Backend ignores duplicates using the client-generated ID.

This ensures no data loss and a smooth experience for field employees.

---

## 4. Explain the difference between SQL and NoSQL databases. Which would you recommend here?

For this application, I would clearly choose **SQL**.

### Why SQL fits this project:
- Strong relationships between users, employees, clients, and check-ins.
- Need for transactions and consistency (e.g., only one active check-in per employee).
- Frequent aggregation queries for dashboards and reports.

### Why not NoSQL here:
- The data model is structured and relational.
- Joins and constraints are important.
- Schema flexibility is not a major requirement.

### My recommendation:
- Use **PostgreSQL** as the primary database.
- Optionally add Redis for caching or fast lookups.
- Keep analytics in SQL unless scale demands a separate system.

---

## 5. What is the difference between authentication and authorization? Where are they implemented?

- **Authentication** answers: *Who are you?*
- **Authorization** answers: *What are you allowed to do?*

**Authentication:**
- Implemented in the login flow.
- JWT is issued after validating credentials.
- Middleware verifies the token and attaches user info to the request.

**Authorization:**
- Role-based checks (e.g., manager-only routes).
- Middleware like `requireManager` protects sensitive endpoints.
- Frontend hides manager UI for non-managers, but backend remains the source of truth.

---

## 6. What is a race condition?

A race condition happens when **multiple requests modify shared data at the same time**, and the result depends on timing.

### Potential race condition here:
- Two check-in requests for the same employee arrive simultaneously.
- Both see no active check-in.
- Both insert a new check-in.

### How I would prevent it:

- Enforce rules at the **database level**, not just in application logic.
- Add a **unique constraint** so an employee can have only one active check-in.
- Catch database constraint errors and return a clean API response.

This guarantees correctness even under heavy concurrency.

---

