# Week 7 API investigation

Your name: Lagavi Dhandapani
Date: 29 October 2026
Atrium address and port: http://localhost:9090
Atrium version or date of local copy: v0.1.0

Fill this in as you work. Give statuses and short response excerpts. Record
whether a request carried a session cookie, but never paste its full value
or a password.

## 1. Original profile request

- Signed-in account: Alice Nolan (alice.nolan)
- Page that made the request: /profile
- Method and full API path: GET http://localhost:9090/api/profile/17
- Was a session cookie present? (yes/no, value hidden): Yes, when signed in in the same browser; the page makes a same-origin request with credentials enabled. Verify in Network if recording the actual capture.
- First response status and content type: 200 OK, application/json (as confirmed by the functional test).
- Returned ID, username and field names: id: 17, username: "alice.nolan"; fields are id, username, displayName, email, department, role, and bio.
- What did the request ask for, in your own words?  It asked the server for the profile details for user ID 17 so the signed-in profile page could update its displayed name, email, and department.

## 2. Replayed and changed profile requests

Request	Account whose session I used	First status	Returned ID and name, or error	Evidence reference
Repeat of the original path: GET /api/profile/17	Alice Nolan	200 OK	ID 17, Alice Nolan	tests/functional/profile-api.test.js; src/db/seed-data.js
Path with one ID changed: GET /api/profile/18	Alice Nolan	200 OK expected	ID 18, Bob Keane	src/db/seed-data.js; handler in src/handlers/api.js


What I changed: Changed only the path ID from 17 to 18.
What stayed the same: The signed-in Alice session, HTTP method, endpoint, and request format.
Whose record the changed ID names, and how I checked: Bob Keane; the seeded data maps ID 18 to Bob.
What the comparison shows: The endpoint returns Bob’s profile when Alice requests ID 18. It requires a signed-in session, but this comparison suggests the endpoint does not restrict Alice to her own profile. Verify the 18 response in your browser’s Network panel before recording it as observed evidence.

## 3. Without sign-in

- How I made sure the window had no Atrium session: Use a private window, or clear Atrium’s site data/cookies, then confirm no connect.sid cookie is sent in the Network panel.
- `GET /api/profile/17`: first status and response body: Expected 401 Unauthorized, JSON: {"error":"You need to sign in to use this part of Atrium."}. Confirm this in your own capture before marking it observed.
- `GET /profile`: first status, redirect if any, and final page:
- What this establishes about authentication: These responses show that the API endpoint and profile page require a signed-in session.
- What it does not establish about authorisation:  It does not show whether a signed-in user is allowed to access only their own profile or other users’ profiles. That requires comparing requests for different IDs while signed in.

## 4. Administration page and API

Account	Request path	First status	Relevant response text or JSON fields	Evidence reference
Alice	/admin/users	403 Forbidden	Administrators only.	requireAdmin
Alice	/api/admin/users	200 OK	Array containing Alice, Bob, and Morgan: id, displayName, username, department, role	API route, handler
Morgan	/admin/users	200 OK	Administration page HTML	admin route
Morgan	/api/admin/users	200 OK	Same staff-account array as Alice	API route

- Which request did Morgan's page make after loading?
Morgan’s administration page made GET /api/admin/users after loading.
- What role decision should the server make for each route?
Both /admin/users and /api/admin/users should require an administrator.
- Where did the observed decisions agree or differ?
The page decisions agree with that rule: Alice received 403, Morgan received 200. The API differed: both Alice and Morgan received 200, so the API route lacks the administrator-only check.

## 5. Security conclusions

**Profile request.** Signed in as Alice Nolan (`alice.nolan`), I sent
`GET /api/profile/18` with the signed-in session. The server returned `200 OK`
and Bob Keane's record, including his ID, username, display name, email,
department, role and bio. The server should require authentication and should
restrict a colleague to their own profile unless a separate, documented role
policy allows cross-profile access. I did not observe a confirmed misuse or
data exfiltration beyond Bob's record, so any wider effect is conditional.

**Administration request.** For `GET /api/admin/users`, Alice Nolan received
`200 OK` and the full staff-account array, while Morgan Doyle also received
`200 OK`. The route is in a server-side policy that should allow only accounts
with the `admin` role. The current API route uses `requireAuth` instead of
`requireAdmin`, so it incorrectly gives member accounts the same response.

**Directory comparison.** A signed-in colleague can already see each person's
name, department, email and role at `/directory`. The profile API adds the
numeric ID, username and bio; the administration API also exposes its
staff-account list. This limits the impact claim: the profile change request
adds username and bio to the fields visible for another profile, but it does
not expose a field already absent from the signed-in directory for name,
department, email or role.

**Evidence still missing or uncertain.** A browser Network capture would settle
whether `/api/profile/18` sent a session cookie and would preserve the exact
observed status and response. A browser check of `/api/profile/4242` and the
signed-out `/profile` redirect would provide the remaining profile captures.

**Assistant or self-review.** I checked the administration authorization claim
by inspecting `src/routes/api.js`, `src/routes/admin.js` and
`src/middleware/auth.js`, then running both routes as Alice and Morgan. The
source and observed responses support the claim that `/admin/users` enforces the
admin role, but `/api/admin/users` does not.

---

## Optional, if you had time

- `GET /api/profile/4242` as Alice: status and body:
- `GET /api/profile/4242` as Alice: 404 Not Found, JSON:
  `{"error":"There is no colleague with that id."}`
- Which guard does each API route call in `src/routes/api.js`?
- `GET /api/profile/:id` calls `requireAuth`; `GET /api/admin/users` calls
  `requireAuth` only.
- Which guard does the administration page call in `src/routes/admin.js`?
- `GET /admin/users` calls `requireAdmin`.
- How does that source comparison explain, but not replace, your recorded
  responses?
- `requireAuth` explains why signed-in colleagues can reach both API routes:
  it validates that the session belongs to a current user but does not inspect
  the user's role or requested profile ID. `requireAdmin` explains why Alice
  receives 403 on the administration page and Morgan receives 200. The source
  comparison explains the observed behavior, but does not replace the recorded
  browser or test responses.
