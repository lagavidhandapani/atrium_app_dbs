# Week 2 security requirements

Your name: Lagavi Dhandapani
Date: 04 October 2026


---

## 1. What this application is

Atrium is a small internal staff workspace. Atrium also has

1. A staff directory
2. Shared Documents
3. Profile page


## 2. What is worth protecting

Four assets. For each one, say what it is and what it would cost if it were seen,
changed or unavailable. Write the cost so that somebody outside the team could understand it.

1. Staff directory records- If the staff directory data are exposed staff can be targeted by phishing and also other people can impersonate with the details.

2. Login credentials- If password and username are exposed the attacker can login with the credentials and can get access to private data.

3. Staff Profile Data- If the data of the staffs get compromised it damages the privacy and trust of the staff.

4. Shared Documents and Internal resources- If these documents are lost, staffs can lose their important work done before.


## 3. The requirements

Four sentences, in your own words. Each one should say what is not allowed and to
whom.

1. Agents with no admin access cannot access the admin credentials or profile
2. Agents cannot login with the credentials of other agent
3. Person without the atrium credentials cannot access atrium directory (Draft from assistant)
4. No agent other than admin can edit other agents personal information and profile (Draft from assistant)


## 4. One I rejected or rewrote

- The original sentence: Anybody can access Atrium without logging in the atrium directory
- My version: A person who have not signed in with the atrium credentials cannot access below details from the directory.
  1. Staff information
  2. Shared informations.
  3. Profile pages of the staff.
- Which test it failed, and why: This test failed because atrium always redirects away the users who has not logged in from the protected pages.

## 5. How somebody would check one of these

Pick one requirement. Write the steps for a person who has never seen Atrium and
cannot ask you anything.

- The requirement:
1. Install Node.js 24 
2. Run the below commands,
npm ci
npm run reset
npm run doctor
npm start
3. Open http://localhost:9090 in the browser

- Sign in as:

Username	Password	Role
alice.nolan	SpringRiver44	staff
bob.keane	CopperLane19	staff
morgan.doyle	QuietHarbour08	administrator

- Steps:

1. http://localhost:9090 
2. Try to open the staff profile page or directory page without logging in.

- What result would mean the requirement is met:

Browser needs to redirect to a login page 

- What result would mean it is not met:

If the page loads and shows the shared informations from the directory or the staff profiles
---

## Optional, if you had time

Your four requirements in order, most important first, with one sentence each on
why it is in that position.

1. Person without the atrium credentials cannot access atrium directory- If it gets exposed it compromises the privacy of the data in the directory.
2. Agents cannot login with the credentials of other agent- If one agent gets credentials of the other agent, privacy of the agent gets exposed.
3. Agents with no admin access cannot access the admin credentials or profile- Every agent have different jobs so only admin users need to have admin credentials. Other agents who have the credentials without the knowledge of how the admin access actually works, it may be lead to misuse of the credentials and also it can create technical issue of they edit anything without the knowledge.
4. No agent other than admin can edit other agents personal information and profile.