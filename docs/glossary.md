# Domain Glossary

Single source of truth for the ubiquitous language used across the product family
(web service, web app, mobile app). Promoted from `document-processor/docs/glossary.md`
plus the new auth / review / export / cost terms.

## Capture & ingestion

| Term | Definition |
|------|------------|
| **Document** | An uploaded image of an invoice, ticket, or payment receipt |
| **Document Type** | Invoice, Ticket, or PaymentReceipt |
| **Pre-processing** | Transcoding HEIC to JPEG/PNG and rasterizing PDF pages before OCR |
| **Capture** | Acquiring a document image via mobile camera or web upload |

## Extraction & validation

| Term | Definition |
|------|------------|
| **Extraction** | The OCR result — raw text output from the OCR engine |
| **Parsed Data** | Structured fields extracted from the raw text (amounts, dates, identifiers, vendor) |
| **Validation** | Regional rule check against Parsed Data (e.g., CUIT format, invoice number pattern) |
| **Validation Result** | Pass/Fail + list of rule violations with reasons |
| **Region Code** | ISO 3166-1 alpha-2 country code driving which validation module runs (e.g., "AR") |
| **Extraction Audit** | Immutable trail recording user, timestamp, provider, confidence, and action per extraction (AR compliance) |
| **Confidence** | OCR/provider score attached to an extraction; logged for observability |

## Identity & lifecycle

| Term | Definition |
|------|------------|
| **Document ID** | UUID v7 assigned at ingestion; used as the stored image filename |
| **Status** | PENDING → OCR_IN_PROGRESS → VALIDATING → COMPLETED / VALIDATION_FAILED / OCR_FAILED |
| **Image Key** | MinIO/S3 object key derived from the content hash (SHA-256) and file extension — content-addressed for dedup + portability |
| **Storage Lifecycle** | Policy that auto-deletes/archives images when storage exceeds configured watermarks and alerts are unattended |
| **High Watermark** | Storage usage percentage (default 85%) that triggers an alert |
| **Critical Watermark** | Storage usage percentage (default 95%) that triggers escalation |
| **Multi-Recipient** | PDF scenario where each page represents a different recipient sharing a common vendor structure |

## Access & security

| Term | Definition |
|------|------------|
| **API Key** | SHA-256 hashed bearer token for machine-client authentication; managed via admin UI/CLI |
| **JWT** | JSON Web Token for web + mobile authentication (rotating refresh tokens) |
| **Refresh Token** | Long-lived rotating credential that mints short-lived access JWTs |
| **Role** | `ADMIN` or `REVIEWER` — drives RBAC middleware decisions |
| **RBAC** | Role-based access control enforced by middleware on protected routes |

## Review & export

| Term | Definition |
|------|------------|
| **Review Queue** | Back-office list of documents pending human confirmation of extracted fields |
| **Manual Fix Flag** | Marker set when a human corrects an extracted field during review |
| **Export** | Serialized, signed output of reviewed document data (deferred: PDF liquidación) |

## Cost & abuse governance

| Term | Definition |
|------|------------|
| **Rate Limit** | Per-API-key/IP sliding window on ingestion and image download |
| **Daily Quota** | 100 docs/day global cap + per-key quota, enforced by the worker before dequeuing |
| **Kill Switch** | Budget-triggered halt of the OCR queue above the daily cap |
| **Dead Letter** | Failed extraction/upload parked with retry count for manual retry |
| **Offline Queue** | Mobile IndexedDB store (≤50 pending) with FIFO + exponential backoff + dead-letter |
