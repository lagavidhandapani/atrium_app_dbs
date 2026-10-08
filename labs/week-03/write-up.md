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
- What came back: I have not checked what the running application returns yet. From reading the code, I think the input will make the SQL condition always true, so the search may return all colleagues
- How this differs from the ordinary case: The ordinary search should show only Bob. The test input may show everyone. Do not write this expected result as something I saw until I try it in the running application

## 5. My answer

Delete the two that do not apply.

**Real** / **Not real** / **Not settled**

**Why, in two or three sentences.** Write for somebody who has not seen any of this.

**What would change my mind.** If new information would alter this answer, say what.

**How far this answer reaches.** What you established applies to a particular page,
a particular set of data and this version of the application. Say what you have
shown, and be careful not to claim more.

## 6. Back to the assistant

The thing you noted in section 2, that you had not verified at the time.

- Did I check it?
- Was it right?

---

## Optional, if you had time

The other two findings matched the same rule. Why are they not the same situation?
Two sentences.
