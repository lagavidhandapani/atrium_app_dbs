# Week 6 evidence record

Your name: Lagavi Dhandapani
Date: 29 October 2026
Atrium version or date of local copy: v0.1.0

Keep this file in `my-work/week-06-write-up.md`. Record what you observed,
including an unexpected result. Do not paste a password or full session cookie.

## 1. Evidence audit

Item	Scanner claim, observed evidence, background or missing evidence?	Where I can find it
Static alert and rule	Historical Semgrep output reports sql-built-by-joining-text on the directory query. This is a scanner claim, not by itself proof of exploitable behavior.	Saved Semgrep output; Week 5 summary in triage notes.
Static code excerpt	The saved output shows the old query joining q into SQL. The current handler instead uses LIKE ? and binds the search pattern, so the saved finding does not describe the current code.	Historical excerpt: saved Semgrep output; current code: src/handlers/directory.js.
Dynamic alert and risk label	Week 5 records Z2 as a reflected HTML-injection concern, but does not preserve its exact ZAP alert name, risk, or confidence. Do not infer those labels from the notes.	Week 5 triage notes, section 2, Z2.
Dynamic request and response	The notes describe an unescaped q in the result summary, but do not include the exact alert payload, matching response, or browser evidence that markup/script executed. The Week 4 SQL-injection request is separate evidence and should not be substituted for this alert.	Code location: views/directory/index.ejs; missing runtime artefact: the relevant ZAP request/response and browser observation.
Signed-in coverage check	Week 4 records Alice’s signed-in /dashboard and directory requests as 200 OK. The recorded search was Alice Nolan, not the exact Alice query requested in the lab.	Week 4 runtime evidence, section 2.
Ordinary comparison	Week 4 records Alice’s ordinary GET /directory?q=Alice as 200 OK, with Alice’s result visible.	Week 4 runtime evidence, section 4.
What is the most important missing artefact for each alert?

Static: The exact source revision or snapshot Semgrep scanned. The saved output contains the old concatenated query, whereas the current handler is parameterized.
Dynamic: The exact ZAP alert evidence for the reflected-output concern: request (including payload), response body, and observed browser behavior. The notes do not preserve enough to verify execution or report a risk/confidence label.

## 2. Static finding

Title: Historical SQL injection finding in the directory search — fixed in the current code

Affected route, file and lines: GET /directory, handled by src/handlers/directory.js. The saved Semgrep output flagged the old query construction at lines 8–11. The route requires sign-in in src/routes/directory.js.

Account and conditions tested: Alice Nolan (alice.nolan), signed in. The evidence covers directory searches on this route only.

Tool output used: The saved Semgrep output reports scan-rules.sql-built-by-joining-text for the old directory query. Week 4 notes record a normal search returning 200 OK and a quote-containing search returning 500 Internal Server Error with a SQLite syntax error. See labs/week-03/saved-scan-output.txt and labs/week-04/runtime-evidence.md.

Code trace: The route requires authentication, then showDirectory reads the q query parameter (current line 21). It passes q to searchDirectory. In the historical code scanned, q was concatenated into SQL; the saved Semgrep excerpt shows that at lines 8–11. In the current code, searchDirectory builds a search pattern and passes it separately to SQL placeholders (lines 7–14). This prevents the input from being interpreted as SQL.

Ordinary request and result: Signed in as Alice, GET http://localhost:9090/directory?q=Alice returned 200 OK; the directory showed Alice’s result. Recorded in Week 4, section 4.

Test request and result: The Week 4 notes record GET http://atrium.localhost:9090/directory?q=Alice+Nolan%27 (decoded query: Alice Nolan') returning 500 Internal Server Error with a SQLite syntax error near “Nolan.” That differed from the ordinary search, which returned 200 OK. After the fix, the regression test for q: "Alice Nolan'" returns 200 OK; see tests/functional/directory.test.js.

My conclusion: The historical code and recorded response support that a quote in the search input could break the directory query. The evidence does not show that data was read or changed. The current code uses bound parameters, and the focused directory tests pass, including the quote-containing search.

Impact within this scope: The tested historical input caused an error on Alice’s signed-in directory search. Any broader effect, such as data disclosure or impact on other routes or accounts, was not established.

Repair and how I would check it: Bind the search pattern using SQL placeholders instead of concatenating it into the SQL text. That repair is now present in src/handlers/directory.js. Check it with ordinary and quote-containing searches; the focused functional tests passed, 6/6.

One other static match: The same Semgrep rule flagged src/handlers/resources.js, but that query uses WHERE resources.id = ? and supplies the ID separately via .get(req.params.id). That match is not evidence of the same injection issue because the outside value is bound, not joined into the SQL.

## 3. Dynamic finding

Title: Reflected search term is rendered without escaping — execution not confirmed

Affected URL and output location: Signed-in GET /directory?q=...; the q value is rendered unescaped in the result summary at line 20 of views/directory/index.ejs. The search input itself uses escaped output at line 13.

Account and conditions tested: Alice Nolan, signed in. Week 4 notes say the ordinary directory search worked. The saved notes do not identify which account was used for the Z2 alert request.

ZAP alert, risk and confidence: Week 5 triage records Z2 as a supported reflected HTML-injection concern. The saved notes do not give the exact ZAP alert name, risk, or confidence, or say whether this was my report or the class report. I need to check the original ZAP alert before recording those details.

Ordinary request and result: GET http://localhost:9090/directory?q=Alice returned 200 OK, and the browser displayed Alice’s result. The saved notes do not include a response excerpt.

Alert request repeated and result: Not recorded in the available evidence. The exact payload, request URL, response body, and browser behavior are missing, so I cannot say whether the alert reproduced or whether any script executed.

Code that explains the result: In views/directory/index.ejs, the result summary inserts q using unescaped EJS output (<%- q %>). The search-box value uses escaped output (<%= q %>) at line 13. This source supports an unescaped-output concern but does not, by itself, prove script execution.

My conclusion: The current template renders the search term unescaped in the result summary, which could allow submitted markup to be interpreted by the browser. The saved evidence does not establish that a specific payload reached this output or executed. I need the original ZAP request and response, plus a browser observation, to confirm the dynamic behavior.

Impact within this scope: Untrusted search text may be interpreted as HTML in the signed-in directory page. Script execution, session impact, and effects on other routes or users have not been established.

Repair and how I would check it: Use escaped EJS output (<%= q %>) in the result summary. Then repeat the recorded harmless test input, verify it appears as text in the response and browser, and check that ordinary searches still work.

## 4. Cross-check and review

What did runtime evidence add to the static finding?
It showed that the historical directory search did more than look risky in the source: a quote-containing search reportedly returned 500 Internal Server Error with a SQLite syntax error, while an ordinary search returned 200 OK. This supports query breakage, but does not show that data was read or changed. The current code has since been changed to use bound parameters.

What did code evidence add to the dynamic finding?
It identified the likely output location: the directory template renders q unescaped in the result summary. The search-box value is escaped. This supports an HTML-injection concern, but does not prove that the alert payload executed.

Who or what reviewed my drafts?
An AI assistant reviewed the drafts against the saved lab evidence and current code.

One specific criticism:
The draft could not claim a confirmed dynamic vulnerability or give the ZAP risk and confidence because the saved notes do not include the alert’s exact request, response, or browser behavior.

What I changed, or why I did not change it:
I limited the conclusion to what the evidence supports and marked the missing ZAP details as needing verification, rather than guessing or treating the source observation as proof of script execution.

## 5. My report draft

Finding reviewed: Reflected search-term output in the signed-in directory page

Check	Present and where, or missing and next action
Plain title and exact location	Present: reflected search-term concern at GET /directory?q=..., in the result summary at line 20 of views/directory/index.ejs.
Account, route and conditions	Route and signed-in condition are identified. The account used for the ZAP alert is missing; check the original ZAP request/history.
Normal and tested result	The ordinary Alice search is recorded as 200 OK. The alert request’s exact payload, status, response and browser result are missing; retrieve them from ZAP before claiming the alert reproduced.
Code or HTTP artefact supporting each claim	The template supports the unescaped-output observation. The matching ZAP request/response and browser behavior are still needed to support a dynamic finding.
Impact limited to the evidence	Present: the draft says markup may be interpreted, but does not claim script execution or session impact was demonstrated.
Repair that addresses the cause	Present: change the result-summary output to escaped EJS (<%= q %>), then retest with the recorded input and an ordinary search.
The next action I will take: Find the original ZAP alert and record its exact request, response, risk/confidence, account, and what the browser displayed. If that evidence is unavailable, keep the finding qualified as a source-code concern rather than a confirmed dynamic result.
