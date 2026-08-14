# Product Specification v1.0

## Document Processor — Product Family

> **Status:** Approved v1.0
> **Date:** 2026-08-13
> **Scope:** cross-component product behavior across the web service, web app, and mobile app
> **Contract:** `docs/contracts/openapi.yaml` (single source of truth)
> **Backend baseline:** `document-processor/docs/spec.md` v1.0 (9 features / 31 scenarios, promoted)

---

## 1. Surfaces

| Surface | Repo | Role |
|---------|------|------|
| Web service | `document-processor` | Ingestion API + async OCR/validation worker + storage lifecycle |
| Web app | `document-processor-web` | Back-office: review, validation queue, API-key admin, dashboard |
| Mobile app | `document-processor-mobile` | Capture/scan with offline queue and progressive sync |

The backend features (F1–F9) are **promoted unchanged** from `document-processor/docs/spec.md`.
This document adds the cross-component journeys and the new family features (auth D9, review,
export, and cost controls D12).

---

## 2. Cross-Component Journeys

### Journey 1 — Capture → Process → Review → Export

```gherkin
Feature: End-to-end document journey
  As a product operator
  I want a document to flow from capture through processing to review and export
  So that extracted data is verified by a human and made usable

  Scenario: Full happy path across all three surfaces
    Given a reviewer is authenticated on the web app
    When a mobile user captures an invoice and it syncs to the web service
    And the OCR pipeline processes it and Argentina validation passes
    Then the document appears in the web app review queue with status COMPLETED
    And the reviewer can export the parsed data (JSON or CSV)

  Scenario: Validation failure routes to the review queue
    Given a document's Argentina validation fails (e.g. invalid CUIT)
    When the worker finishes processing
    Then the document has status VALIDATION_FAILED
    And it appears in the web app review queue for manual fix
```

### Journey 2 — Offline capture with progressive sync

```gherkin
Feature: Offline-first mobile capture
  As a mobile user in a low-connectivity environment
  I want captured documents to queue locally and sync when connectivity returns
  So that no capture is lost

  Scenario: Capture while offline then sync
    Given the mobile device is offline
    When the user captures an invoice
    Then it is stored in the local queue (IndexedDB, ≤50 pending)
    And when connectivity returns, documents upload in FIFO order
    And the app polls status until each reaches a terminal state
```

---

## 3. Web App Features

### Feature: Authentication (JWT)

```gherkin
Feature: Authentication
  As a web app user
  I want to log in with a username and password
  So that I can access my documents and the review queue

  Scenario: Successful login
    Given a user with role REVIEWER exists
    When they log in with valid credentials
    Then they receive an access token and a refresh token
    And subsequent requests use the Bearer access token

  Scenario: Failed login
    Given a user enters an incorrect password
    When they attempt to log in
    Then the response status is 401

  Scenario: Access token refresh
    Given a user has a valid refresh token
    When their access token expires
    Then they can exchange the refresh token for a new pair
    And the old refresh token is rotated
```

### Feature: Document browsing

```gherkin
Feature: Document browsing
  As a reviewer
  I want to list, filter, and inspect documents
  So that I can find documents needing attention

  Scenario: List with filters
    Given documents with mixed statuses exist
    When the reviewer filters by status=VALIDATION_FAILED
    Then only matching documents are returned, paginated

  Scenario: View document detail
    Given a document with parsed_data exists
    When the reviewer opens its detail view
    Then parsed fields, confidence, and validation_result are shown
```

### Feature: Review queue and manual fix

```gherkin
Feature: Manual review
  As a reviewer
  I want to correct and approve extracted fields
  So that bad extractions don't propagate downstream

  Scenario: Approve a document with corrected fields
    Given a document is in the review queue
    When the reviewer edits parsed fields and approves
    Then the document is marked reviewed with the corrected fields persisted

  Scenario: Request changes
    Given a document is in the review queue
    When the reviewer requests changes with a comment
    Then the document is flagged for re-extraction
```

### Feature: Export

```gherkin
Feature: Export
  As an operator
  I want to export a document's extracted data
  So that it can be consumed by other systems

  Scenario: Export JSON
    Given a COMPLETED document
    When the operator requests export
    Then a flattened JSON payload is returned

  Scenario: Export CSV
    Given a COMPLETED document
    When the operator requests export with Accept: text/csv
    Then a CSV payload is returned
```

### Feature: API key administration (ADMIN)

```gherkin
Feature: API key administration
  As an ADMIN
  I want to create, list, and revoke machine-client API keys
  So that I can manage programmatic access

  Scenario: Create an API key
    Given an ADMIN is authenticated
    When they create a key with a label
    Then the raw key is shown once and stored only as a SHA-256 hash

  Scenario: Revoke an API key
    Given an ADMIN is authenticated
    When they revoke a key by prefix
    Then that key is rejected on subsequent requests (403)
```

### Feature: Dashboard

```gherkin
Feature: Dashboard
  As an operator
  I want a summary of processing activity
  So that I can monitor throughput and failures

  Scenario: Dashboard summary
    Given documents exist in various statuses
    When the operator opens the dashboard
    Then counts by status and recent activity are shown
```

---

## 4. Mobile App Features

### Feature: Camera capture

```gherkin
Feature: Camera capture
  As a mobile user
  I want to capture a document with the camera
  So that it can be submitted for processing

  Scenario: Capture meets quality requirements
    Given the camera captures at minimum 1920×1080
    And the resulting image is JPEG ≤5 MB
    When the user captures a document
    Then it is accepted and queued for upload

  Scenario: Capture below quality threshold
    Given the camera captures below minimum resolution
    When the user captures a document
    Then a quality warning is shown and the capture is rejected
```

### Feature: Offline queue

```gherkin
Feature: Offline queue
  As a mobile user
  I want an offline queue with retry and dead-letter
  So that captures survive connectivity loss and are retried safely

  Scenario: FIFO upload with exponential backoff
    Given the device is online
    When queued documents upload
    Then they upload in FIFO order
    And failed uploads retry with exponential backoff

  Scenario: Dead-letter after max retries
    Given a document fails upload repeatedly
    When retries exceed the maximum
    Then it is moved to a dead-letter state and surfaced for manual retry
```

---

## 5. Cost & Abuse Guardrails (D12)

```gherkin
Feature: Cost controls
  As a system operator
  I want ingestion bounded by rate limits and a daily quota
  So that a public API cannot drive unbounded cost

  Scenario: Within daily quota
    Given the global daily count is below the cap
    When a client ingests a document
    Then it is accepted (202)

  Scenario: Daily quota exceeded
    Given the global daily cap of 100 documents has been reached
    When a client ingests a document
    Then the response is 429 with a Retry-After header

  Scenario: Per-key rate limit
    Given a client has submitted 60 requests in the current minute
    When they submit another
    Then the response is 429
```

---

## 6. Feature Summary

| # | Feature | Surface |
|---|---------|---------|
| F1–F9 | Promoted backend features (ingestion, OCR, validation, rate limit, auth, storage) | Web service |
| F10 | User authentication (JWT login/refresh) | Web app + mobile |
| F11 | Document browsing | Web app |
| F12 | Review queue + manual fix | Web app |
| F13 | Export | Web app |
| F14 | API key administration | Web app |
| F15 | Dashboard | Web app |
| F16 | Camera capture | Mobile |
| F17 | Offline queue + dead-letter | Mobile |
| F18 | Cost controls (daily quota) | Web service |

---

**Status:** Approved v1.0.
