# Week 8 remediation record

Your name: Lagavi Dhandapani
Date: 17 November 2026
Atrium version or copy used: Atrium v0.1.0

## 1. Original finding and before evidence

- Page and route: Staff directory: `GET /directory`
- Account used: Alice Nolan (`alice.nolan`)
- Earlier finding or source I started from: I used the quote-character test in
  `tests/functional/directory.test.js`. I also checked the directory code. I
  did not have a browser capture from the old version.
- Ordinary search text, URL, status and results-table rows: I searched for
  `Keane` at `/directory?q=Keane`. It gave status `200 OK`. It showed Bob
  Keane, Finance, `bob.keane@atrium.example`, and member.
- No-match search text, URL, status and results-table rows: I searched for
  `Zawadzki` at `/directory?q=Zawadzki`. It gave status `200 OK`. It showed
  zero rows and a no-results message.
- Comparison search text, URL, status and results-table rows: I searched for
  `Alice Nolan' OR 1=1 --`. The URL was
  `/directory?q=Alice%20Nolan%27%20OR%201%3D1%20--`. It gave status `200 OK`.
  It showed zero rows.
- What the comparison shows, and what it does not show: It shows that the
  comparison text was treated as search text. It does not prove that the old
  copy was vulnerable.

## 2. Root cause

- Source of the search text, with file and line: The search box sends `q` at
  `views/directory/index.ejs:13`. The handler reads `req.query.q` at
  `src/handlers/directory.js:23`.
- Path through the handler: `showDirectory` sends `q` to `searchDirectory`.
  `searchDirectory` adds `%` to the text for a LIKE search.
- Database operation, with file and line: In
  `src/handlers/directory.js:11-15`, the SQL uses `?` placeholders. SQLite
  receives the search text as a separate value twice.
- Control missing on that path: No SQL control is missing in the current copy.
  The search text is not added to the SQL sentence.
- Difference between the visible result and the root cause: The comparison text
  gave no rows because it was read as normal search text. This matches the fixed
  SQL and bound values in the current code.

## 3. Security property and useful behaviour

- One sentence stating the property the repair must enforce: Search text must
  be a value, not part of the SQL command.
- Ordinary behaviour that must remain, including name and department search:
  An empty search lists all colleagues. `Keane` finds Bob Keane by name.
  `Finance` finds Bob Keane by department. A no-match search shows no results.

## 4. Candidate fixes

| Candidate | Where the control acts | Effect on the property and ordinary search | Decision |
|---|---|---|---|
| Use `?` placeholders and bound values | In `searchDirectory` | Keeps the SQL fixed. Name and department searches still work. | Selected |
| Remove or escape quote characters | Before the SQL command | This may block valid text. It is easy to get wrong. | Rejected |

- Assistant used, if any: I used Copilot to check the code and run tests.
- One suggestion I changed or rejected, and why: I rejected removing quote
  characters. Search text should be data, not SQL code.
- Reason for my choice: The current code already uses `?` placeholders and
  `.all(pattern, pattern)`. This keeps SQL safe and normal searches working.

## 5. Change made

- File and function changed: I made no application change. The function is
  `searchDirectory` in `src/handlers/directory.js`.
- Changed lines or a short before and after code extract: No code changed.
  The SQL is:
  `"WHERE display_name LIKE ? OR department LIKE ?"`,
  followed by `.all(pattern, pattern)`.
- How the change keeps search text separate from the database instruction:
  The SQL sentence stays fixed. SQLite receives the search text as a separate
  bound value.
- What I left outside this repair: I did not change profiles, administration,
  sign-in, or document text.

## 6. First checks after the change

- Same ordinary search: I searched for `Keane`. Status was `200 OK`. Bob Keane
  was the only result.
- Same no-match search: I searched for `Zawadzki`. Status was `200 OK`. There
  were zero rows.
- Same comparison search: I searched for `Alice Nolan' OR 1=1 --`. Status was
  `200 OK`. There were zero rows.
- Empty search result: I used `q=`. Status was `200 OK`. All three colleagues
  were listed.
- `npm test` result and any failing test: 23 tests passed. 0 tests failed.
- `npm run scan` result, if available: The scan found warnings at
  `src/db/reset.js:12-13` and `src/handlers/resources.js:1-4`. These are fixed
  SQL strings joined by text. This is not a directory-search result.

## 7. Handoff to Week 9

- What this evidence supports: The current directory search keeps search text
  separate from SQL. Name, department, empty, no-match, and comparison searches
  worked as expected.
- Cases or related paths still to check: Check an old copy against this copy.
  Also check `%` and `_` in search text, browser output, and other search paths.
- A focused automated test I would add: Request
  `/directory?q=Alice%20Nolan%27%20OR%201%3D1%20--`. Check status `200`, zero
  rows, and no colleague listed.
- Any known regression or remaining risk: This is not a before-and-after result.
  The directory page prints `q` using `<%- q %>` at
  `views/directory/index.ejs:20`. It should be checked for HTML output.
  Resource descriptions also need review.

## 8. Document description output

- Resource changed for the temporary check: In a temporary database, I changed
  resource ID `1` to:
  `<img src=x onerror="document.body.dataset.xss=1">`.
- List and detail page before the repair: `/resources` and `/resources/1`
  returned `200 OK`. Both pages showed the raw image markup.
- Rule for printing a description as page content: Descriptions should appear
  as text, not HTML. If HTML is allowed, use a clear allowed-HTML policy or
  sanitizer. Otherwise, escape the text with `<%=`.
- Two view files and lines changed: No repair was made. Raw output is at
  `views/resources/index.ejs:13` and `views/resources/show.ejs:13`.
- List and detail page after the repair: Not checked because no repair was made.
- Ordinary description checked after restoring the seed data: Not done. The
  temporary database was not used as the main Atrium database.
- What this check does not establish: It proves that raw markup was printed.
  It does not prove that a browser ran the script. It does not check every
  browser or security header.
