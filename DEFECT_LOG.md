# Master Defect Log (Conduit RealWorld Application)

**Student Name:** Israt Jahan  
**Course Assignment:** KSA 02 - Software QA Defect Discovery, Code Remediation & Presentation  
**Target Application:** Conduit (React / Express.js / Sequelize / PostgreSQL)  

---

## Executive Summary
This document contains the Master Defect Log for the Conduit RealWorld Application. During software quality assurance (SQA) exploratory and static analysis testing, **5 genuine, non-trivial defects** were discovered across frontend component state management, service parameter formatting, backend user authentication, form validation, and rendering security.

---

## Defect Index

1. **BUG-01:** JWT Token Signed with Incomplete User Payload on Login
2. **BUG-02:** Premature Promise Resolution and State Reset in Comment Editor
3. **BUG-03:** Malformed Query String Ampersands in Articles API Service
4. **BUG-04:** Blank / Whitespace-Only Article Submission Bypasses Backend Validation
5. **BUG-05:** Unsanitized Markdown Render Vulnerability / Raw HTML Execution

---

### BUG-01: JWT Token Signed with Incomplete User Payload on Login

* **Defect ID:** `BUG-01`
* **Title:** User Authentication JWT Token Payload Missing Username Attribute on Sign-In
* **Severity:** Critical
* **Priority:** High
* **Component:** Backend Controller (`backend/controllers/users.js`)
* **Environment:** Node.js v18+, Express.js v4.x, Windows 11 / Linux, Chrome / Edge
* **Pre-conditions:**
  1. User account must already exist in the database (e.g., registered user).
  2. Server must be running with valid `JWT_KEY` environment variable configured.

#### Steps to Reproduce:
1. Send a POST request to `/api/users/login` with payload `{ "user": { "email": "user@example.com", "password": "password123" } }`.
2. Inspect the HTTP response body returned from the server containing `{ user: { email, token, ... } }`.
3. Decode the JWT token string using `jwt.decode(token)` or at [jwt.io](https://jwt.io).
4. Observe the decoded token payload claims.

* **Expected Result:**
  The JWT token payload should contain complete claims for authenticated user identification: `{ "username": "johndoe", "email": "user@example.com", "iat": ... }`.
* **Actual Result:**
  The JWT token payload contains an `undefined` username claim: `{ "username": undefined, "email": "user@example.com", "iat": ... }` because `jwtSign(user)` was passed `req.body.user` (which lacks the `username` field) instead of the database user instance `existentUser`.
* **Proof of Defect Note:**
  Capture screenshot of decoded JWT token at jwt.io showing `"username": undefined` alongside Postman/REST client response.

---

### BUG-02: Premature Promise Resolution and State Reset in Comment Editor

* **Defect ID:** `BUG-02`
* **Title:** Form Input Textarea Cleared Before Async Comment Submission Completes
* **Severity:** High
* **Priority:** Medium
* **Component:** Frontend React Component (`frontend/src/components/CommentEditor/CommentEditor.jsx`)
* **Environment:** Web Browsers (Chrome, Firefox, Edge, Safari), React 18
* **Pre-conditions:**
  1. User is authenticated and viewing an individual article page (`/article/:slug`).
  2. Comment section is visible.

#### Steps to Reproduce:
1. Type a multi-paragraph comment into the comment text area.
2. Simulate a slow network connection (e.g., Throttle to 3G in Chrome DevTools) or trigger a backend error (e.g., temporary server offline).
3. Click the **"Post Comment"** button.
4. Observe the text input box immediately following the button click.

* **Expected Result:**
  The input text should remain in the textarea while the HTTP request is pending and should only reset to empty string `""` upon successful API response (`201 Created`). If an error occurs, the typed text must be preserved.
* **Actual Result:**
  The textarea text vanishes instantly upon clicking submit because `.then(setForm({ body: "" }))` calls `setForm` synchronously during render rather than passing a callback function `() => setForm({ body: "" })`. If the API request fails, user content is permanently lost.
* **Proof of Defect Note:**
  Record screen video showing form text disappearing immediately on button click before network request completes.

---

### BUG-03: Malformed Query String Ampersands in Articles API Service

* **Defect ID:** `BUG-03`
* **Title:** Articles Service Uses Double Ampersands (`&&`) in HTTP Query Parameters
* **Severity:** High
* **Priority:** High
* **Component:** Frontend Service (`frontend/src/services/getArticles.js`)
* **Environment:** Web Browsers, Axios HTTP Client, Express Query Parser
* **Pre-conditions:**
  1. Frontend application navigating between global feed, tag feed, or profile tabs.

#### Steps to Reproduce:
1. Open Browser Developer Tools -> Network Tab.
2. Navigate to Home page or click on any tag / pagination link.
3. Inspect outgoing GET request URLs dispatched by Axios (e.g., `/api/articles?tag=react&&limit=3&&offset=0`).
4. Inspect backend parameter parsing for `req.query`.

* **Expected Result:**
  URL parameters should be separated by single ampersands `&` (e.g., `/api/articles?tag=react&limit=3&offset=0`), allowing Express to parse `req.query.limit` and `req.query.offset`.
* **Actual Result:**
  URL parameters contain double ampersands `&&`, causing Express middleware to parse parameter keys as `&limit` and `&offset`. As a result, `req.query.limit` and `req.query.offset` are `undefined`, breaking custom limit/offset pagination.
* **Proof of Defect Note:**
  Capture screenshot of Chrome DevTools Network tab showing URL with `&&limit=3&&offset=0` and backend console log of `req.query`.

---

### BUG-04: Blank / Whitespace-Only Article Submission Bypasses Backend Validation

* **Defect ID:** `BUG-04`
* **Title:** Article Creation Accepts Pure Whitespace Title/Description/Body
* **Severity:** Medium
* **Priority:** Medium
* **Component:** Backend Controller (`backend/controllers/articles.js`)
* **Environment:** Node.js / Express backend API
* **Pre-conditions:**
  1. User is logged in with valid bearer auth token.

#### Steps to Reproduce:
1. Send a POST request to `/api/articles` with body `{ "article": { "title": "   ", "description": "   ", "body": "   ", "tagList": [] } }`.
2. Observe backend response status code and database entry created.

* **Expected Result:**
  The server should reject requests with whitespace-only strings and return HTTP `422 Unprocessable Entity` with message `"A title is required"`.
* **Actual Result:**
  The server accepts the request (`HTTP 201`), passes `if (!title)` check (since non-empty string is truthy), and creates an article with empty slug `""` or invalid database state.
* **Proof of Defect Note:**
  Capture Postman request/response showing HTTP 201 Created response for payload containing only spaces.

---

### BUG-05: Unsanitized Markdown Render Vulnerability / Raw HTML Execution

* **Defect ID:** `BUG-05`
* **Title:** Article View Renders Raw HTML Elements Without Sanitization
* **Severity:** High
* **Priority:** Medium
* **Component:** Frontend View (`frontend/src/routes/Article/Article.jsx`)
* **Environment:** Web Browsers, `markdown-to-jsx` Library
* **Pre-conditions:**
  1. Malicious user publishes an article containing embedded HTML markup.

#### Steps to Reproduce:
1. Create an article with body content: `<img src="invalid" onerror="alert('XSS Risk')">` or inline `<script>`/`<iframe>` tags.
2. Navigate to the article viewing page `/article/:slug`.
3. Observe rendered DOM output.

* **Expected Result:**
  HTML tags should be sanitized or rendered as plain text escaped entities, preventing inline event handling or script execution.
* **Actual Result:**
  `markdown-to-jsx` renders raw HTML elements directly into the DOM tree (`forceBlock: true`), exposing the application to Stored XSS vectors if malicious HTML payloads are stored.
* **Proof of Defect Note:**
  Capture screenshot of rendered article page displaying executed inline HTML elements or JavaScript alert dialog.
