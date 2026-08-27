# Code Fix Walkthrough & Technical Remediation (`FIXES.md`)

**Student Name:** Israt Jahan  
**Course Assignment:** KSA 02 - Software QA Defect Discovery, Code Remediation & Presentation  
**Target Application:** Conduit (RealWorld App)  

---

## Executive Overview
This document details the code-level remediation implemented directly within the local workspace repository. Two major non-trivial defects (**BUG-01: JWT Token Payload Missing Username** and **BUG-03: Malformed Query String Ampersands**) were selected for full root-cause analysis, source code modification, code diff creation, and verification testing.

---

## 1. Remediation Detail: BUG-01 (JWT Payload Defect)

### 1.1 Root Cause Analysis
In the authentication controller [`backend/controllers/users.js`](file:///c:/Users/israt/Downloads/saq%20project/conduit-realworld-example-app/backend/controllers/users.js#L41-L57), the `signIn` function processes user login requests:

```javascript
// BEFORE REMEDIATION (Line 51)
const { user } = req.body; // user is { email, password }
...
existentUser.dataValues.token = await jwtSign(user);
```

The helper utility `jwtSign(payload)` in [`backend/helper/jwt.js`](file:///c:/Users/israt/Downloads/saq%20project/conduit-realworld-example-app/backend/helper/jwt.js#L4-L9) expects a payload object with both `username` and `email`:
```javascript
module.exports.jwtSign = async (payload) => {
  return jwt.sign(
    { username: payload.username, email: payload.email },
    privateKey
  );
};
```
Because `user` originates directly from `req.body.user` (which only contains client-submitted `email` and `password`), `user.username` evaluates to `undefined`. As a result, every login token generated had claims `{ username: undefined, email: "user@example.com" }`. When downstream middleware or microservices attempt to identify users by JWT `username`, authorization fails.

### 1.2 Code Patch Application
The argument passed to `jwtSign` was changed from `user` (`req.body.user`) to `existentUser` (the Sequelize database model retrieved via `User.findOne`).

#### Code Diff (`backend/controllers/users.js`)
```diff
@@ -48,7 +48,7 @@
     const pwd = await bcryptCompare(user.password, existentUser.password);
     if (!pwd) throw new ValidationError("Wrong email/password combination");
 
-    existentUser.dataValues.token = await jwtSign(user);
+    existentUser.dataValues.token = await jwtSign(existentUser);
 
     res.json({ user: existentUser });
   } catch (error) {
```

### 1.3 Verification & Regression Testing Steps
1. **Login Test:** Issue POST request to `/api/users/login` with registered credentials.
2. **Payload Inspection:** Copy the returned `token` string and decode it via JWT decoder.
3. **Assertion:** Confirm the decoded payload contains valid `username` (e.g., `{ "username": "israt_jahan", "email": "israt@example.com" }`).
4. **Regression Check:** Issue subsequent authenticated requests (`GET /api/user`, `POST /api/articles`) using `Authorization: Token <jwt>`. Verify server returns `200 OK` / `201 Created` with correct user context.

---

## 2. Remediation Detail: BUG-03 (Malformed API Query Parameters)

### 2.1 Root Cause Analysis
In the frontend service module [`frontend/src/services/getArticles.js`](file:///c:/Users/israt/Downloads/saq%20project/conduit-realworld-example-app/frontend/src/services/getArticles.js#L5-L13), HTTP GET requests for article feeds were constructed using double ampersands (`&&`) as parameter delimiters:

```javascript
// BEFORE REMEDIATION (Lines 7-13)
const url = {
  favorites: `api/articles?favorited=${username}&&limit=${limit}&&offset=${page}`,
  feed: `api/articles/feed?limit=${limit}&&offset=${page}`,
  global: `api/articles?limit=${limit}&&offset=${page}`,
  profile: `api/articles?author=${username}&&limit=${limit}&&offset=${page}`,
  tag: `api/articles?tag=${tagName}&&limit=${limit}&&offset=${page}`,
};
```

Standard URI specifications (RFC 3986) mandate single ampersands (`&`) between query parameters. Due to `&&`, Express HTTP query parser produced keys starting with leading ampersands (e.g. `req.query['&limit']` and `req.query['&offset']`). Consequently, backend logic in `articles.js` reading `req.query.limit` always defaulted to `3` and ignored offset pagination calculations.

### 2.2 Code Patch Application
All occurrences of double ampersands (`&&`) in the `url` map were replaced with single ampersands (`&`).

#### Code Diff (`frontend/src/services/getArticles.js`)
```diff
@@ -5,11 +5,11 @@
 async function getArticles({ headers, limit = 3, location, page = 0, tagName, username }) {
   try {
     const url = {
-      favorites: `api/articles?favorited=${username}&&limit=${limit}&&offset=${page}`,
-      feed: `api/articles/feed?limit=${limit}&&offset=${page}`,
-      global: `api/articles?limit=${limit}&&offset=${page}`,
-      profile: `api/articles?author=${username}&&limit=${limit}&&offset=${page}`,
-      tag: `api/articles?tag=${tagName}&&limit=${limit}&&offset=${page}`,
+      favorites: `api/articles?favorited=${username}&limit=${limit}&offset=${page}`,
+      feed: `api/articles/feed?limit=${limit}&offset=${page}`,
+      global: `api/articles?limit=${limit}&offset=${page}`,
+      profile: `api/articles?author=${username}&limit=${limit}&offset=${page}`,
+      tag: `api/articles?tag=${tagName}&limit=${limit}&offset=${page}`,
     };
```

### 2.3 Verification & Regression Testing Steps
1. **Network Audit:** Open Chrome/Firefox DevTools Network Tab.
2. **Pagination Test:** Click page `2` on Global Feed pagination or click on a popular tag filter.
3. **Query Inspection:** Inspect outgoing HTTP GET request URL. Confirm request string is `/api/articles?limit=3&offset=1`.
4. **Backend Log Verification:** Check backend terminal/console log for `req.query`. Confirm `limit: "3"` and `offset: "1"` are parsed into separate object properties.
5. **UI Assertion:** Verify correct offset slice of articles is fetched and rendered in the DOM.

---

## 3. Test Suite Execution Output

Automated regression suite was executed via `vitest`:
```bash
npx vitest --run
```
* **Result:** All unit test suites passed cleanly with 0 failures, confirming no regressions introduced into helpers or components.
