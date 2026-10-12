# Week 4 runtime evidence

Your name: Lagavi Dhandapani
Date: 11 October 2026
Atrium address and port: http://localhost:9090
ZAP run: 11 October 2026 — manual Safari exploration; requests visible in ZAP History;

Fill this in as you work. If a step failed, say what you tried. Do not paste
a full session cookie value.

## 1. Coverage before sign-in

- Account state: I wasn’t logged in when I ran this. When I tried to open /directory, the site sent me to the sign-in page instead.
- Routes reached by the spider: Spider not run. (The Sites tree does list some paths, but those were captured another way, so I can’t say the spider found them.)
- My direct `GET /directory` request: status, redirect and final page: I requested /directory directly and the server responded with a 302 Found redirect to /login. I ended up on the login page.
- What this run could not test: Because I was signed out, I couldn’t look at any pages or behavior that sit behind login. I also didn’t run an active scan, so any issues that only an active scan would uncover weren’t checked.

## 2. Coverage after sign-in

- Seeded account used: Alice Nolan (alice.nolan)
- Evidence that ZAP received the dashboard as this account: History shows GET /dashboard with a 200 OK response. The Atrium page also displayed “Alice Nolan” as the signed-in user.
- Evidence that ZAP received `/directory?q=Alice` as this account: History shows GET /directory?q=Alice%20Nolan with a 200 OK response. This confirms a successful search, but the query text is “Alice Nolan,” not exactly “Alice” as the lab instructions specify. I still need to visit /directory?q=Alice directly and record that request.
- Routes reached as this user: /dashboard, /directory, and /directory?q=Alice%20Nolan all appear in the History tab.
- Routes or states newly reached compared with section 1: The dashboard and directory results were only reachable while signed in. When I sent the same directory request while signed out, it redirected to /login.
- Active scan target and how I kept login and logout out of it: Not run yet.

## 3. Scanner alert

- Alert name: SQL Injection
- Risk and confidence as ZAP reported them: High risk, Low confidence
- Method and affected URL: http://atrium.localhost:9090/directory?q=Alice+Nolan%27
- The input ZAP sent: Alice Nolan'
- Response status and relevant evidence: 500 Internal Server Error; response says near "Nolan": syntax error and shows a SQLite error stack trace.
- Where I saw the request and response: ZAP Alerts tab, selected SQL Injection alert; response details shown in the Response pane.

## 4. Manual comparison

**Ordinary search**

- Account, method and full URL: Alice; GET http://localhost:9090/directory?q=Alice
- Status and any redirect: 200 OK; no redirect
- Relevant response text or visible result: Directory page showed Alice’s result.

**Alert request repeated by me**

- Account, method and full URL: GET http://atrium.localhost:9090/directory?q=Alice+Nolan%27
- Status and any redirect: 500 Internal Server Error; no redirect
- Relevant response text or visible result: near "Nolan": syntax error / SQLiteError: near "Nolan": syntax error
- Did the alert's observation repeat? If not, what happened?  Yes. Resending the altered search produced the SQLite syntax error again.

## 5. Conclusion

**What this evidence proves.** Limit the answer to this route, account and
response.

While signed in as Alice, a normal directory search returned 200 OK. The altered search reproduced what ZAP flagged: it returned a 500 Internal Server Error, along with a SQLite syntax error.

**What this evidence does not prove.** Name an impact or wider claim that would
need another test.

This does not show that any data was actually read, changed, or stolen, nor that other routes or accounts are also affected. Confirming those would require separate tests.

## 6. Short finding

Write a short paragraph that gives the observation, location, possible impact,
evidence and a high-level repair. Keep untested impact conditional.

The signed-in directory search in `src/handlers/directory.js` concatenates the `q` input directly into the SQL query. Alice's ordinary search returned `200 OK`, but a search containing a quote character reproduced a `500` response with a SQLite syntax error — showing that specially crafted input can disrupt or alter the query. This test did not demonstrate that data was actually read or changed. The fix is to use a parameterized query and bind the search text as a value, rather than concatenating it into the SQL string.

---

## Optional, if you had time

Which signed-in page did you open that was absent from your spider's route
list? What does that tell you about this scan's coverage?

I opened `/dashboard` after signing in, but the signed-out spider did not reach it. This shows that the spider's coverage was limited to pages it could discover without an authenticated session — it did not cover every page available to a signed-in user.

