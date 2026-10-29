# Week 6 evidence record

Your name: Lagavi Dhandapani
Date: 29 October 2026
Atrium version or date of local copy: v0.1.0

Keep this file in `my-work/week-06-write-up.md`. Record what you observed,
including an unexpected result. Do not paste a password or full session cookie.

## 1. Evidence audit

Item	Classification	What the evidence shows	Where to find it
Static alert and rule	Scanner claim	Historical Semgrep output flags sql-built-by-joining-text on the directory query. This is the scanner’s claim and does not prove exploitable behavior.	Saved Semgrep output; Week 5 summary in triage notes
Static code excerpt	Background	The saved output shows the old query concatenating q into the SQL. The current handler uses LIKE ? and binds the search pattern, so the saved finding does not describe the current code.	Historical: saved Semgrep output. Current: src/handlers/directory.js
Dynamic alert and risk label	Missing evidence	Week 5 records Z2 as a reflected HTML-injection concern, but it does not preserve the exact ZAP alert name, risk, or confidence. These labels must not be inferred from the notes.	Week 5 triage notes, section 2, Z2
Dynamic request and response	Missing evidence	The notes describe an unescaped q in the result summary. They do not include the alert payload, the matching response, or browser evidence that markup or script executed. The Week 4 SQL-injection request is separate evidence and must not be substituted for this alert.	Code location: views/directory/index.ejs. Missing: the relevant ZAP request/response and browser observation
Signed-in coverage check	Observed evidence (partial)	Week 4 records Alice’s signed-in /dashboard and directory requests as 200 OK. The recorded search was Alice Nolan, not the exact Alice query the lab requested.	Week 4 runtime evidence, section 2
Ordinary comparison	Observed evidence	Week 4 records Alice’s ordinary GET /directory?q=Alice as 200 OK, with Alice’s result visible.	Week 4 runtime evidence, section 4

What is the most important missing artefact for each alert?

- Static: The exact source revision or snapshot that Semgrep scanned. The saved output shows the old concatenated query, while the current handler is parameterized, so without the scanned revision the finding cannot be tied to any specific code state.
- Dynamic:The ZAP evidence for the reflected-output concern: the request with its payload, the response body, and the observed browser behavior. Without these, the notes cannot verify that anything executed or support a risk or confidence label.

## 2. Static finding

Title: Historical SQL injection finding in the directory search, fixed in the current code

Affected route, file and lines: GET /directory, handled by src/handlers/directory.js. The saved Semgrep output flagged the old query construction at lines 8–11. The route requires sign-in, enforced in src/routes/directory.js.

Account and conditions tested: Alice Nolan (alice.nolan), signed in. The evidence covers directory searches on this route only.

Tool output used: The saved Semgrep output reports scan-rules.sql-built-by-joining-text on the old directory query. The Week 4 notes record a normal search returning 200 OK and a search containing a quote returning 500 Internal Server Error with a SQLite syntax error. Sources: labs/week-03/saved-scan-output.txt and labs/week-04/runtime-evidence.md.

Code trace:

The route requires authentication.
showDirectory then reads the q query parameter (current line 21) and passes it to searchDirectory.
In the historical code that was scanned, q was concatenated directly into the SQL text. The saved Semgrep excerpt shows this at lines 8–11.
In the current code, searchDirectory builds a search pattern and passes it separately to SQL placeholders (lines 7–14). Because the value is bound rather than joined into the query, the input can no longer be interpreted as SQL.

Ordinary request and result: Signed in as Alice, GET http://localhost:9090/directory?q=Alice returned 200 OK and the directory showed Alice’s result (Week 4, section 4).

Test request and result: The Week 4 notes record GET http://atrium.localhost:9090/directory?q=Alice+Nolan%27 (decoded query: Alice Nolan'). It returned 500 Internal Server Error with a SQLite syntax error near “Nolan”, which differed from the ordinary search’s 200 OK. After the fix, the regression test for q: "Alice Nolan'" returns 200 OK; see tests/functional/directory.test.js.

My conclusion: The historical code and the recorded response support the claim that a quote in the search input could break the directory query. The evidence does not show that any data was read or changed. The current code uses bound parameters, and the focused directory tests pass, including the quote-containing search.

Impact within this scope: The tested historical input caused an error in Alice’s signed-in directory search. Any broader effect, such as data disclosure or impact on other routes or accounts, was not established.

Repair and how I would check it: Bind the search pattern with SQL placeholders instead of concatenating it into the SQL text. This repair is now present in src/handlers/directory.js. To check it, run ordinary and quote-containing searches. The focused functional tests passed, 6/6.

One other static match: The same Semgrep rule also flagged src/handlers/resources.js. That query uses WHERE resources.id = ? and supplies the ID separately via .get(req.params.id). This match is not evidence of the same injection issue, because the outside value is bound and not joined into the SQL.

## 3. Dynamic finding

Title: Reflected search term is rendered without escaping, execution not confirmed

Affected URL and output location: Signed-in GET /directory?q=.... The q value is rendered unescaped in the result summary at line 20 of views/directory/index.ejs. The search input itself uses escaped output at line 13.

Account and conditions tested: Alice Nolan, signed in. The Week 4 notes say the ordinary directory search worked. The saved notes do not identify which account was used for the Z2 alert request.

ZAP alert, risk and confidence: Week 5 triage records Z2 as a supported reflected HTML-injection concern. The saved notes do not give the exact ZAP alert name, risk, or confidence, and they do not say whether this was my report or the class report. I need to check the original ZAP alert before recording any of these details.

Ordinary request and result: GET http://localhost:9090/directory?q=Alice returned 200 OK, and the browser displayed Alice’s result. The saved notes do not include a response excerpt.

Alert request repeated and result: Not recorded in the available evidence. The payload, request URL, response body, and browser behavior are all missing, so I cannot say whether the alert reproduced or whether any script executed.

Code that explains the result: In views/directory/index.ejs, the result summary inserts q using unescaped EJS output (<%- q %>). The search-box value uses escaped output (<%= q %>) at line 13. This supports an unescaped-output concern but does not, by itself, prove script execution.

My conclusion: The current template renders the search term unescaped in the result summary, which could allow submitted markup to be interpreted by the browser. The saved evidence does not establish that a specific payload reached this output or executed. To confirm the dynamic behavior, I need the original ZAP request and response, plus a browser observation.

Impact within this scope: Untrusted search text may be interpreted as HTML on the signed-in directory page. Script execution, session impact, and effects on other routes or users have not been established.

Repair and how I would check it: Switch the result summary to escaped EJS output (<%= q %>). Then repeat the recorded harmless test input, verify that it appears as plain text in both the response and the browser, and confirm that ordinary searches still work.

## 4. Cross-check and review

What did runtime evidence add to the static finding?
It showed that the historical directory search did more than look risky in the source. According to the Week 4 notes, a search containing a quote returned 500 Internal Server Error with a SQLite syntax error, while an ordinary search returned 200 OK. This supports the claim that the query could be broken by input, but it does not show that any data was read or changed. The current code has since been changed to use bound parameters.

What did code evidence add to the dynamic finding?
It identified the likely output location: the directory template renders q unescaped in the result summary, while the search-box value is escaped. This supports an HTML-injection concern, but it does not prove that the alert payload executed.

Who or what reviewed my drafts?
An AI assistant reviewed the drafts against the saved lab evidence and the current code.

One specific criticism:
The draft could not claim a confirmed dynamic vulnerability, or state the ZAP risk and confidence, because the saved notes do not include the alert’s exact request, response, or browser behavior.

What I changed, or why I did not change it:
I limited the conclusion to what the evidence supports. I marked the missing ZAP details as needing verification instead of guessing, and I did not treat the source-code observation as proof of script execution.

## 5. My report draft

Finding reviewed: Reflected search-term output in the signed-in directory page

Check	Status	Where it is present, or the next action
Plain title and exact location	Present	Reflected search-term concern at GET /directory?q=..., in the result summary at line 20 of views/directory/index.ejs.
Account, route and conditions	Partly present	The route and the signed-in condition are identified. The account used for the ZAP alert request is missing. Next action: check the original ZAP request and history.
Normal and tested result	Partly present	The ordinary Alice search is recorded as 200 OK. The alert request’s payload, status, response and browser result are missing. Next action: retrieve them from ZAP before claiming the alert reproduced.
Code or HTTP artefact supporting each claim	Partly present	The template supports the unescaped-output observation. The matching ZAP request and response, plus browser behavior, are still needed to support a dynamic finding.
Impact limited to the evidence	Present	The draft says markup may be interpreted. It does not claim that script execution or session impact was demonstrated.
Repair that addresses the cause	Present	Change the result-summary output to escaped EJS (<%= q %>), then retest with the recorded input and an ordinary search.

Next action: Locate the original ZAP alert and record its exact request, response, risk and confidence, account, and what the browser displayed. If that evidence cannot be found, keep the finding qualified as a source-code concern and do not present it as a confirmed dynamic result.
