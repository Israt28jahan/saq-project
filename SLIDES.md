# Presentation Deck Script & Slide Content (`SLIDES.md`)

**Student Name:** Israt Jahan  
**Course Assignment:** KSA 02 - SQA Presentations (with Slide Deck)  
**Target Application:** Conduit (RealWorld App) — Defect Discovery & Code Remediation  

---

## Slide 1: Title & Student Information

* **Header:** End-to-End Software QA Audit & Code Remediation
* **Sub-header:** Case Study on Conduit RealWorld Web Application
* **Presenter:** Israt Jahan
* **Course:** KSA 02 - Software Quality Assurance & Testing
* **Date:** Academic Term 2026

### Slide Content:
* **Target Project:** Conduit Fullstack Application
* **Scope of Work:**
  * Architectural QA Analysis & Exploratory Testing
  * Discovery & Documentation of 5 Critical/High/Medium Defects
  * Production-Grade Source Code Remediation (2 Bugs)
  * Automated & Manual Verification Plan

### Speaker Notes:
> "Good day everyone. My name is Israt Jahan. Today I am presenting my Software QA audit and code remediation project on Conduit, a fullstack RealWorld example application built with React, Express, and PostgreSQL. In this presentation, I will walk you through our QA methodology, the master defect log, two deep-dive code fixes applied directly to the codebase, and our engineering takeaways."

---

## Slide 2: Target Application Architecture & Tech Stack

* **Header:** Target Application Architecture & Tech Stack
* **Sub-header:** RealWorld Modular Fullstack Architecture

### Slide Content:

```
┌─────────────────────────────────────────────────────────────┐
│                      FRONTEND LAYER                         │
│   React 18  │  Vite  │  React Router v6  │  Axios Client    │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTP / REST API
┌──────────────────────────────▼──────────────────────────────┐
│                       BACKEND LAYER                         │
│   Node.js  │  Express.js  │  JWT Auth  │  Bcrypt Encryption │
└──────────────────────────────┬──────────────────────────────┘
                               │ ORM Interface
┌──────────────────────────────▼──────────────────────────────┐
│                      DATABASE LAYER                         │
│           Sequelize ORM  │  PostgreSQL Database             │
└─────────────────────────────────────────────────────────────┘
```

* **Frontend:** React 18 SPA (Vite builder, Context API state management, standard CSS)
* **Backend REST API:** Node.js, Express framework, JWT token authentication
* **Database & ORM:** Sequelize ORM with relational PostgreSQL mapping

### Speaker Notes:
> "To perform effective QA, we first mapped the architecture. Conduit uses a decoupled architecture: a React 18 SPA consuming a Node.js Express REST API backed by Sequelize ORM. Understanding this data flow between Axios services, Express controllers, and Sequelize models was key to discovering edge-case state and security defects."

---

## Slide 3: QA Strategy & Testing Methodology

* **Header:** QA Strategy & Testing Methodology
* **Sub-header:** Multi-Tier Quality Assurance Framework

### Slide Content:
1. **Exploratory & Edge-Case Testing:**
   * Form input boundary testing (empty strings, whitespace-only, special characters)
   * UI state transition testing (promise handling, state clearing)
2. **Security & Authentication Audit:**
   * JWT claim verification & payload completeness
   * Cross-Site Scripting (XSS) analysis on markdown renderers
3. **API & Parameter Syntax Audit:**
   * HTTP query string formatting and Express middleware parameter parsing
4. **Automated Unit Testing:**
   * Vitest test suite execution for regression detection

### Speaker Notes:
> "Our QA strategy combined manual exploratory testing with static code inspection and automated testing. We focused heavily on boundaries, input validation, authentication token payloads, and state synchronization issues between client promises and backend models."

---

## Slide 4: Master Defect Summary Table

* **Header:** Master Defect Catalog (5 Discovered Defects)
* **Sub-header:** Comprehensive SQA Findings

### Slide Content:

| Bug ID | Defect Title | Component | Severity | Priority | Status |
|---|---|---|---|---|---|
| **BUG-01** | JWT Token Missing Username Claim on Login | Backend (`users.js`) | **Critical** | **High** | **REMEDIATED** |
| **BUG-02** | Textarea Cleared Before Async Post Completes | Frontend (`CommentEditor.jsx`) | **High** | **Medium** | Logged |
| **BUG-03** | Malformed `&&` Query Strings Break Pagination | Frontend (`getArticles.js`) | **High** | **High** | **REMEDIATED** |
| **BUG-04** | Whitespace-Only Inputs Bypass Article Validation | Backend (`articles.js`) | **Medium** | **Medium** | Logged |
| **BUG-05** | Unsanitized Markdown Renders Raw HTML Elements | Frontend (`Article.jsx`) | **High** | **Medium** | Logged |

### Speaker Notes:
> "Here is our Master Defect Log summary table. We logged 5 genuine, non-trivial defects. Notice the severity distribution: 1 Critical, 3 High, and 1 Medium. We selected BUG-01 and BUG-03 for immediate code-level remediation."

---

## Slide 5: Deep Dive: Bug Fix 1 (JWT Payload Defect)

* **Header:** Deep Dive: Bug Fix 1 — User Auth JWT Payload
* **Sub-header:** BUG-01 Remediation in `backend/controllers/users.js`

### Slide Content:

* **Defect Description:** User login JWT token signed payload `{ "username": undefined, "email": "user@example.com" }`.
* **Root Cause:** `jwtSign(user)` was receiving `req.body.user` (which lacks `username`), instead of the database user instance.
* **Code Remediation Diff:**

```diff
- existentUser.dataValues.token = await jwtSign(user);
+ existentUser.dataValues.token = await jwtSign(existentUser);
```

* **Outcome & Impact:**
  * Restored complete JWT payload claims (`username` + `email`).
  * Resolves downstream authorization failures on protected API routes.

### Speaker Notes:
> "In Bug Fix 1, `jwtSign` was being called with `req.body.user` which only contained email and password. By changing the argument to `existentUser`, the signed JWT now includes the actual username fetched from PostgreSQL. This fixed critical authentication breakdowns across protected endpoints."

---

## Slide 6: Deep Dive: Bug Fix 2 (Malformed Query Syntax)

* **Header:** Deep Dive: Bug Fix 2 — Malformed API Query Parameters
* **Sub-header:** BUG-03 Remediation in `frontend/src/services/getArticles.js`

### Slide Content:

* **Defect Description:** Article pagination and filtering failed because query strings used `&&` instead of `&`.
* **Root Cause:** `api/articles?tag=react&&limit=3&&offset=0` caused Express query parser to parse keys as `&limit` and `&offset`, leaving `req.query.limit` undefined.
* **Code Remediation Diff:**

```diff
- favorites: `api/articles?favorited=${username}&&limit=${limit}&&offset=${page}`,
+ favorites: `api/articles?favorited=${username}&limit=${limit}&offset=${page}`,
```

* **Outcome & Impact:**
  * Express server now correctly receives `req.query.limit` and `req.query.offset`.
  * Article feed pagination and tag filtering now operate seamlessly.

### Speaker Notes:
> "In Bug Fix 2, the frontend API service used double ampersands `&&` to join query parameters. Express parsed this as literal parameter names with leading ampersands, breaking offset pagination. Replacing `&&` with single `&` restored proper pagination behavior."

---

## Slide 7: Verification & Regression Testing Evidence

* **Header:** Verification & Regression Testing Evidence
* **Sub-header:** Rigorous Quality Assurance Validation

### Slide Content:
* **Manual Verification Results:**
  * Decoded JWT token verified via `jwt.decode` -> Contains valid `username` & `email`.
  * Outgoing HTTP request verified via Browser DevTools -> `/api/articles?limit=3&offset=1`.
* **Automated Unit Testing Execution:**
  * Command: `npx vitest --run`
  * Test Suite Status: **PASSED (100% Success, 0 Failures)**
* **Regression Status:**
  * No breaking changes or regressions introduced to adjacent components or API routes.

### Speaker Notes:
> "To ensure our fixes did not introduce regressions, we performed both manual API/UI verification and ran the automated Vitest suite. All test suites passed cleanly with zero errors."

---

## Slide 8: Key Engineering Takeaways & SQA Best Practices

* **Header:** Key Engineering Takeaways & SQA Best Practices
* **Sub-header:** SQA Guidelines for Fullstack Applications

### Slide Content:
1. **Never Trust Client Payloads for Token Generation:**
   * Always construct JWT claims from validated database entities, not raw `req.body` objects.
2. **Strict Parameter & Schema Validation:**
   * Sanitize and strip whitespace strings before performing truthiness checks.
   * Adhere strictly to RFC 3986 URI parameter standards.
3. **Promise-Aware Component State Management:**
   * Avoid executing state setter functions directly inside `.then()` handlers without callback wrappers.
4. **Defensive UI Rendering:**
   * Always sanitize markdown and user-generated content before injection into the DOM.

### Speaker Notes:
> "In conclusion, this QA exercise highlights four major software engineering principles: never trust raw client inputs for JWT signing, enforce RFC standards on HTTP parameters, manage async state callbacks cleanly, and sanitize raw HTML inputs. Thank you for your time!"
