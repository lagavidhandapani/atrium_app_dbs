# Week 5 cross-tool triage

Your name: Lagavi Dhandapani
Date: October 12 2026
Atrium address and port: http://127.0.0.1:9090
Atrium version or date of local copy: v0.1.0 (package.json, local copy)

Fill this in as you work. If a check could not be completed, record what stopped you and what evidence you used instead. Do not paste a full session cookie value.

## 1. Scan coverage and sources

- Static evidence: my `npm run scan` output:

Ran npm run scan. Semgrep reported 3 findings under sql-built-by-joining-text: src/handlers/directory.js:8–11, src/handlers/resources.js:1–5, and src/db/reset.js:12–13. The rule reports SQL text assembled by joining strings; I’ll check whether each query includes outside input or uses bound parameters before deciding what the finding means.

- Runtime evidence: my ZAP run, Week 4 record, or direct browser checks:

ZAP History showed signed-in requests to /dashboard and /directory. The ordinary directory search returned 200 OK. Repeating the altered search GET /directory?q=Alice+Nolan%27 returned 500 Internal Server Error with a SQLite syntax error. ZAP reported a SQL Injection alert for the search; its risk was High and confidence Low.

- Signed-in account used for protected routes: Alice Nolan (alice.nolan)

- Evidence that a protected route returned its page rather than `/login`:

ZAP History showed GET /dashboard and a directory search returning 200 OK; the browser displayed Alice’s dashboard and directory results.

- Active scan target and how login and logout stayed outside it, if I used ZAP:

The scan target was `GET http://atrium.localhost:9090/directory?q=Alice%20Nolan`, run with Alice selected as the user and Recurse unchecked. ZAP reported 289 requests and 3 new alerts, with the visible traffic targeting the directory search. I reviewed the displayed requests and did not see any `/login` or `/logout` calls among them. The `/logout` exclusion was added before this scan.

- An entry my scan did not reproduce, and how I checked it instead:

Z5 did not appear in the results of my directory-search scan. Instead, I checked the `POST /login` response directly in ZAP's History: it returned `302 Found`, and its `Set-Cookie` header for `connect.sid` did not include the `HttpOnly` flag. I did not record the actual cookie value.

## 2. The queue

For each entry, use **supported**, **ruled out**, or **needs review**. Keep overlap separate from status. Name a file and line, or a request and response, in the evidence column. If evidence is missing, say what check would settle it.

| ID | Status | Evidence or missing check | Overlap with another ID? | Investigate now, later, or close? |
|---|---|---|---|---|
| **S1** | Supported | `src/handlers/directory.js` builds its SQL by joining the search value `q` directly into the query string, instead of passing it as a bound parameter. | **Z1** — Z1 is the runtime evidence for this same code path. | **Now**, alongside Z1. |
| **S2** | Ruled out | `src/handlers/resources.js` uses a parameterized query (`WHERE resources.id = ?`), with the ID supplied separately via `.get(req.params.id)`. This rules out the reported concatenation issue. | None found. | **Close.** |
| **S3** | Ruled out | `src/db/reset.js` uses `?` placeholders for its INSERT statement, with seed values passed separately to `.run(...)`. This rules out the reported concatenation issue. | None found. | **Close.** |
| **Z1** | Supported | Both ZAP and a manual replay of the modified directory search returned a `500 Internal Server Error` with a SQLite syntax error near "Nolan." This confirms the input can break the query — it does not show that data was actually read or altered. | **S1** — same code path; Z1 is the runtime side, S1 the source-code side. | **Now**, alongside S1. |
| **Z2** | Supported | `views/directory/index.ejs` renders `q` without escaping it in the search-result summary, supporting a reflected HTML-injection concern. Whether a script would actually run has not been demonstrated — that part is still unproven. | **Z3** — same page and input, different output location. | **Now** — check the alert's exact response and confirm whether the payload is live. |
| **Z3** | Ruled out | In the same file, the search value placed in the input field's `value` attribute uses escaped output (`<%=`), ruling out injection at that specific point. | Related to Z2 (same page/input), but a different, escaped output location. | **Close** this specific concern; double-check the alert's returned HTML if needed. |
| **Z4** | Supported | A ZAP History screenshot shows that a signed-in `GET /dashboard` request returned `200 OK` with no `Content-Security-Policy` header present. | None found. | **Later** — decide on a suitable CSP policy. |
| **Z5** | Supported | A ZAP History screenshot shows `POST /login` returning `302 Found`, with its `connect.sid` cookie set without the `HttpOnly` flag. The cookie's actual value was not recorded. | None found with the directory-search findings. | **Now** — assess the exposure this creates, particularly together with Z2. |

## 3. First three matters to investigate

Rank the matters. Combine entries only when they concern one underlying path. For each, record the relevant account, route or file, possible consequence, uncertainty and next check.

1. **First:**
   - Queue ID or IDs: S1 and Z1 — the source and runtime evidence concern the same `/directory` search path.
   - Account and place: Alice Nolan (`alice.nolan`), signed-in `GET /directory?q=...`; `src/handlers/directory.js`.
   - Why first: The search value is joined directly into a SQL query, and a quote in the request produced a SQLite syntax error. This confirms query breakage, but not that data was read or changed; altered query results or unintended disclosure remain possible.
   - Next check: Verify the search with a bound-parameter implementation, including ordinary searches and a search term containing a quote, and confirm malformed input no longer causes a SQL error.
2. **Second:**
   - Queue ID or IDs: Z2.
   - Account and place: Alice Nolan (`alice.nolan`), signed-in `GET /directory?q=...`; the result summary in `views/directory/index.ejs`.
   - Why second: The summary renders `q` without escaping, so attacker-controlled markup may be interpreted in Alice's page. Script execution is not yet demonstrated; if possible, its impact could be greater because Z5 indicates the session cookie lacks `HttpOnly`.
   - Next check: Replay the recorded alert input with a harmless execution marker in a test session, inspect the response and rendered page, then escape the summary output and verify the marker is displayed as text.
3. **Third:**
   - Queue ID or IDs: Z5.
   - Account and place: Alice Nolan (`alice.nolan`), `POST /login` response setting `connect.sid`; `src/config/session.js`.
   - Why third: The observed cookie lacks `HttpOnly`, and the session configuration explicitly sets `httpOnly: false`. This could let page script read the session cookie, but the practical impact depends on script execution and the cookie's deployment context.
   - Next check: Confirm the cookie attributes in the browser after login and assess the intended cookie policy; enable `HttpOnly` and verify the response cookie no longer exposes the session to page scripts.

## 4. One exclusion or combination

- Entry ID: S2
- Close, exclude from active work, or combine with another entry? :Close the reported SQL-concatenation concern. Do not merge it with another entry.
- Evidence for this decision: In src/handlers/resources.js, the query uses WHERE resources.id = ?, and showResource passes req.params.id separately to .get(...). This means the route ID is bound as a value rather than joined into the SQL text. This only rules out the concatenation issue reported for this specific lookup.
- What change would make me reopen it: The query or route is later changed to concatenate outside input into the SQL, or new runtime evidence shows the ID can unexpectedly alter the query’s structure or results.

## 5. One static and runtime comparison

- Semgrep entry and ZAP entry: S1 and Z1
- What the code observation shows: In src/handlers/directory.js, the /directory search builds its SQL by concatenating the search term q directly into the query string.
- What the request and response show: While signed in as Alice Nolan, a modified GET /directory?q=... request containing a quote character returned 500 Internal Server Error with a SQLite syntax error. An ordinary search, by contrast, returned 200 OK.
- Do they concern the same underlying matter? Why? Yes. Both point to the same /directory search input — S1 shows how the query is built in the code, while Z1 shows how the running application actually responds when that input is altered.
- One claim they do not yet prove: They do not prove that an attacker can retrieve or modify data.


## 6. One finding for a colleague

Write a short paragraph. Include what and where, the account and condition, possible impact, evidence references and a repair direction. Keep untested impact conditional.

**Finding:**

With Alice Nolan (alice.nolan) signed in, submitting a modified search to GET /directory?q=... triggered a 500 Internal Server Error with a SQLite syntax error. The /directory handler concatenates the search term directly into its SQL query, which could allow unintended query behavior. Data access or modification has not been demonstrated. The fix is to bind the search term as a parameter and confirm that searches containing quotes no longer produce SQL errors.

**Evidence references:**

S1 (src/handlers/directory.js, query construction); Z1 (modified directory-search request and the resulting SQLite error response).
---

## Optional, if you had time

What does `src/config/session.js` show about the session that the Z5 response cannot? What does the response confirm that source inspection alone cannot?

`src/config/session.js` shows the configured session-cookie policy: `httpOnly: false`, `sameSite: false`, and `secure: false`. This tells us what the application is set up to request — not necessarily what a specific response actually sent.

The Z5 `POST /login` response confirms this at runtime: in that observed login, the `connect.sid` cookie was set without the `HttpOnly` flag. Source inspection alone cannot confirm which cookie attributes are actually emitted only the running application can show that.