# Week 3 record

Your name: Lagavi Dhandapani
Date: 08 October 2026

Fill this in as you work, rather than at the end. Where you are unsure, write
that you are unsure and say why. That is worth more than a confident sentence you
cannot support.

---

## 1. The finding I chose

State which of the three the tool reported.

- File: directory.js
- Line: 8,9,10,11
- What the tool said about it: This SQL statement is built by joining pieces of text together. If any of those pieces
          came from outside the application, the database will read it as part of the command   
          rather than as a value. 

## 2. What an assistant told me

Say which assistant you asked, and what it said in a sentence or two.

- Assistant used: GitHub Copilot

- Its explanation, in your own words: The search takes the text in the URL's `q` part and puts it straight into an SQL command. Because of this, input that looks like SQL might change the command, and the app may not treat it as normal search text.
The scanner found a real risk of SQL injection. But the report alone does not show what an attacker could see or change.

- One thing it asserted that I had not verified at that point: Someone could write special search text that changes the SQL query. I have not yet tested this on the running application.

## 3. What the code shows

Answer all four. If you cannot answer one, say so.

**Where does the data come from?**
It comes from the q part of the directory URL. For example: /directory?q=Keane

**What happens to it on the way?**
The handler only checks that q is text (a string). Then it sends it to searchDirectory.

**Where does it become dangerous?**
searchDirectory joins q directly into the SQL LIKE conditions and then runs the query. Because of this, input that looks like SQL may become part of the query.

**What stands in the way?**
You must be signed in to open /directory. But the search does not use a bound parameter, so nothing protects against SQL injection at that point.

## 4. What the running application shows

Record both. A single result on its own proves nothing.

**Ordinary case**

- What I entered: I signed in and searched the directory for Keane.
- What came back: Bob Keane appeared. Alice Nolan did not. The existing functional test checks this result.

**The case I was testing for**

- What I entered: I signed in and searched for ' OR 1=1 --
- What came back: All 3 colleagues appeared: Alice Nolan, Bob Keane and Morgan Doyle.
- How this differs from the ordinary case: Searching Keane returned only Bob, and Alice did not appear. This input returned everyone, including Morgan Doyle, an admin.

## 5. My answer

Delete the two that do not apply.

**Real** / **Not real** / **Not settled**

**Why, in two or three sentences.** Write for somebody who has not seen any of this.

The directory search puts the q value straight into an SQL query. When I searched ' OR 1=1 -- , the search returned all 3 colleagues instead of none or one. This shows that my input changed the query.

**What would change my mind.** If new information would alter this answer, say what.

A control search using text that matches no real record should return zero results. If it instead returns all colleagues, the earlier result is not caused by the SQL injection payload, and the finding would need to be reconsidered.

**How far this answer reaches.** What you established applies to a particular page,
a particular set of data and this version of the application. Say what you have
shown, and be careful not to claim more.

## 6. Back to the assistant

The thing you noted in section 2, that you had not verified at the time.

- Did I check it?  Yes. I ran the test in the running application.

- Was it right? Partly. The code suggested the input could change the query, and the application returned all 3 colleagues. That matches, but it only shows the result of this one input.

---

## Optional, if you had time

The other two findings matched the same rule. Why are they not the same situation?
Two sentences.

The reset-code query builds its SQL from fixed text and passes the actual values in separately through `?` placeholders — the user's input never gets joined into the query itself. The resource query works the same way: it uses fixed SQL with a `?` placeholder for the resource ID. The directory search is different it puts the search text straight into the SQL command, which is why that one is risky and these two are not.
