<div align="center">

```
 _                _  __          _
| |    ___   __ _| |/ /_ __  ___| |_
| |   / _ \ / _` | ' /| '_ \/ _ \ __|
| |__| (_) | (_| | . \| | | | (_) | |_
|_____\___/ \__, |_|\_\_| |_|\___/ \__|
             |___/
```

# logknot-core

**Immutable Audit Logging Engine — Go + AWS Serverless**

[![Build](https://img.shields.io/github/actions/workflow/status/YOUR_USERNAME/logknot-core/test.yml?branch=main&label=build&style=flat-square)](https://github.com/YOUR_USERNAME/logknot-core/actions)
[![Go Version](https://img.shields.io/badge/Go-1.22+-00ADD8?style=flat-square&logo=go&logoColor=white)](https://go.dev/)
[![License: MIT](https://img.shields.io/badge/License-MIT-22c55e?style=flat-square)](./LICENSE)
[![Status](https://img.shields.io/badge/Status-Pre--Launch-f59e0b?style=flat-square)]()
[![Go Report Card](https://goreportcard.com/badge/github.com/YOUR_USERNAME/logknot-core?style=flat-square)](https://goreportcard.com/report/github.com/YOUR_USERNAME/logknot-core)

*The backend engine powering [logknot.com](https://logknot.com)*

</div>

---

> **Pre-Launch Engineering Notice**
> This repository contains the core backend architecture and engineering foundation
> for LogKnot, currently under active development ahead of private beta.
> The architecture, IAM policies, and SDK interfaces documented here reflect the
> intended production design. Implementation progresses through the phases tracked
> in the [Roadmap](#roadmap) below.

---

## Table of Contents

- [Engineering Overview](#engineering-overview)
- [System Architecture](#system-architecture)
- [Event Lifecycle — Data Flow](#event-lifecycle--data-flow)
- [Immutability: IAM Enforcement](#immutability-iam-enforcement)
- [Go SDK](#go-sdk)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [Local Development](#local-development)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Security](#security)
- [Related Repository](#related-repository)
- [License](#license)

---

## Engineering Overview

`logknot-core` is the serverless backend that provides tamper-proof, append-only audit
log storage, real-time anomaly detection, and on-demand compliance evidence generation
for multi-tenant B2B SaaS applications.

**Core engineering decisions:**

- **Immutability is an infrastructure guarantee, not an application promise.** AWS IAM
  deny policies on all Lambda execution roles make it structurally impossible to modify
  or delete a stored event — no code path, regardless of privilege level, can bypass this.
- **ULID keys over UUIDs.** Event IDs use the ULID format (`oklog/ulid`), which is
  lexicographically sortable by time. This eliminates the need for a timestamp-based
  secondary index for chronological queries.
- **DynamoDB Streams as the event pipeline.** All downstream processing — anomaly
  detection, archival, metering — is decoupled from the ingest path via Streams.
  Ingest latency is never affected by downstream failures.
- **Cursor-based pagination over offset.** The query API uses DynamoDB's
  `LastEvaluatedKey` as a cursor token, ensuring consistent pagination performance at
  any scale without full-table scans.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                    │
│                                                                         │
│   ┌──────────────┐   ┌─────────────────┐   ┌────────────────────────┐ │
│   │   Go SDK     │   │  React Dashboard│   │   Stripe Billing       │ │
│   │  pkg.go.dev  │   │  Vite + Cognito │   │  Checkout + Webhooks   │ │
│   └──────┬───────┘   └────────┬────────┘   └──────────┬─────────────┘ │
└──────────┼─────────────────────┼────────────────────────┼──────────────┘
           │                     │                        │
┌──────────▼─────────────────────▼────────────────────────▼──────────────┐
│                          EDGE LAYER                                     │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │            AWS API Gateway (REST)                               │  │
│   │     API Key Auth · Usage Plans · WAF · Rate Limiting (429)      │  │
│   └──────────────────────┬──────────────────────────────────────────┘  │
│                          │                                              │
│   ┌──────────────────────┴──────────────────────────────────────────┐  │
│   │            Amazon Cognito                                        │  │
│   │     Dashboard JWT Auth · Email + Password · Session Mgmt        │  │
│   └─────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼───────────────────────────────────────┐
│                        COMPUTE LAYER  (Go Lambdas)                      │
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────────────┐ │
│  │  Ingest Lambda  │  │  Query Lambda   │  │  Export Lambda         │ │
│  │                 │  │                 │  │                        │ │
│  │ • Schema valid  │  │ • Filter/paginate│  │ • CSV (encoding/csv)  │ │
│  │ • ULID assign   │  │ • Cursor-based  │  │ • PDF (gofpdf)        │ │
│  │ • Batch write   │  │ • GSI queries   │  │ • Pre-signed S3 URL   │ │
│  │ POST /v1/events │  │ GET /v1/events  │  │ POST /v1/exports      │ │
│  └────────┬────────┘  └────────┬────────┘  └──────────┬────────────┘ │
│           │                    │                        │              │
│  ┌────────▼────────────────────▼────────────────────────▼───────────┐ │
│  │                    Streams Lambda                                  │ │
│  │                                                                    │ │
│  │   Rule 1: Bulk delete  >50 DELETE actions / 60s from one actor    │ │
│  │   Rule 2: Privilege escalation  action contains "admin"           │ │
│  │   Rule 3: Off-hours access  outside 07:00-22:00 project TZ        │ │
│  │                                                                    │ │
│  │   Triggered by:  DynamoDB Streams (NEW_IMAGE)                     │ │
│  │   Output:        SNS Topic → SES Email → Project Owner            │ │
│  │   Failure path:  SQS Dead Letter Queue                            │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │   Stripe Webhook Lambda    Billing Metering Lambda (EventBridge)  │  │
│  │   Key Manager Lambda       Nightly Usage Counter Lambda           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────┐
│                          STORAGE LAYER                                   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  DynamoDB — Events Table                                         │   │
│  │                                                                   │   │
│  │  PK: projectId      SK: eventId (ULID, lexicographic time-sort)  │   │
│  │  GSI-1: actorId     GSI-2: resourceType                         │   │
│  │  PITR: Enabled      Streams: NEW_IMAGE    Encryption: SSE-S3    │   │
│  │                                                                   │   │
│  │  ┌──── IAM DENY POLICY — applied to all Lambda roles ────────┐  │   │
│  │  │  "Effect": "Deny"                                         │  │   │
│  │  │  "Action": ["dynamodb:UpdateItem","dynamodb:DeleteItem"]  │  │   │
│  │  │  "Resource": "arn:aws:dynamodb:*:*:table/logknot-events-*"│  │   │
│  │  └───────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────┐  ┌────────────────────┐  ┌────────────────────┐  │
│  │ DynamoDB — Keys  │  │  S3 — Exports      │  │  S3 Glacier        │  │
│  │                  │  │                    │  │                    │  │
│  │ SHA-256 key hash │  │ PDF / CSV objects  │  │ 7-year archive     │  │
│  │ planTier         │  │ Pre-signed URLs    │  │ HIPAA retention    │  │
│  │ usageCounter     │  │ TTL: 15 minutes    │  │ Auto-tiered        │  │
│  └──────────────────┘  └────────────────────┘  └────────────────────┘  │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────────────┐
│                         OUTPUT LAYER                                     │
│                                                                          │
│  ┌───────────────┐  ┌──────────────────┐  ┌─────────────────────────┐  │
│  │  AWS SES      │  │  Webhook POST    │  │  CloudWatch + SNS       │  │
│  │  Anomaly      │  │  HMAC-signed     │  │  Lambda error rate >1%  │  │
│  │  alert email  │  │  Customer HTTPS  │  │  DynamoDB throttles     │  │
│  └───────────────┘  └──────────────────┘  └─────────────────────────┘  │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  CloudFront + S3 — Dashboard CDN                                  │  │
│  │  React + Vite static build · Global edge delivery                │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Event Lifecycle — Data Flow

```
Client Application
    │
    │  POST /v1/events
    │  Body: { actor, action, resource, metadata }
    │  Header: X-API-Key: <sha256-hashed-key>
    ▼
API Gateway
    │  1. API key lookup → Keys table
    │  2. Usage plan check → 429 if over limit
    │  3. WAF inspection → block SQLi / XSS / bots
    ▼
Ingest Lambda (Go)
    │  1. JSON body → EventSchema struct
    │  2. go-playground/validator → required fields check
    │  3. oklog/ulid → generate time-sortable eventId
    │  4. dynamodb.BatchWriteItem → up to 25 events per call
    │  5. Return HTTP 201 + { eventId, status: "created" }
    ▼
DynamoDB Events Table
    │  Append-only enforced by IAM Deny (see below)
    │  PITR: continuous backup to any second within 35 days
    │
    ├─────────────────────────────────┐
    ▼                                 ▼
HTTP 201 returned              DynamoDB Streams
to calling client              NEW_IMAGE record emitted
                                       │
                               Streams Lambda (Go)
                                       │
                      ┌────────────────┼─────────────────┐
                      ▼                ▼                  ▼
                 Anomaly?         S3 Glacier          Billing
                SNS → SES          archive            counter
              alert email         (nightly)         (EventBridge)
```

---

## Immutability: IAM Enforcement

Immutability in LogKnot is **not** an application-level policy. It is a structural
constraint enforced by AWS IAM before any code executes.

The following policy statement is attached to every Lambda execution role in the stack —
including the ingest Lambda, the admin tooling, and any future role added to the account:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceAuditImmutability",
      "Effect": "Deny",
      "Action": [
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/logknot-events-*"
    }
  ]
}
```

**Why an explicit `Deny` is unbypassable:**
AWS IAM policy evaluation follows a strict precedence order. An explicit `Deny`
statement overrides any `Allow` granted elsewhere — in the same policy, in another
attached policy, in a role trust policy, or in a resource-based policy. There is
no way to grant `UpdateItem` or `DeleteItem` on the events table when an explicit
`Deny` is in effect, regardless of the caller's other privileges.

This means:
- A developer with full DynamoDB admin access cannot modify an event
- A Lambda function with a misconfigured permission cannot modify an event
- A compromised AWS access key cannot modify an event
- The policy itself can only be changed by modifying IAM, which is separately audited

**Verification:** Any AWS auditor can confirm this property by inspecting the IAM
policy attached to the Lambda execution role. The statement is not buried in code —
it is declared in `template.yaml` and visible in the AWS Console under
IAM → Roles → `logknot-ingest-role-*` → Permissions.

**Additional immutability layers:**

| Layer | Mechanism | Purpose |
|---|---|---|
| DynamoDB PITR | Continuous backup | Restore to any second within 35 days |
| S3 Glacier archive | Nightly EventBridge Lambda | 7-year retention for HIPAA |
| DynamoDB encryption | SSE-S3 (default) | Encryption at rest |
| TLS 1.2+ | API Gateway enforced | Encryption in transit |
| CloudTrail | AWS account-level | Audit trail of IAM and API operations |

---

## Go SDK

> Implementation begins in Phase 3. Interface is frozen for review.

**Installation:**

```bash
go get github.com/YOUR_USERNAME/logknot-core/sdk/go
```

**Usage:**

```go
package main

import (
    logknot "github.com/YOUR_USERNAME/logknot-core/sdk/go"
)

func main() {
    // Initialise once at application startup.
    // Safe for concurrent use across goroutines.
    client := logknot.New("lk_your_api_key")
    defer client.Flush() // Flush remaining buffer on shutdown

    // Log any user action — returns immediately, never blocks.
    err := client.Log(logknot.Event{
        Actor: logknot.Actor{
            ID:   "user_123",
            Type: "user",
        },
        Action: "document.delete",
        Resource: logknot.Resource{
            ID:   "doc_456",
            Type: "document",
        },
        Metadata: map[string]any{
            "ip":     "203.0.113.45",
            "reason": "user_requested",
        },
    })
    if err != nil {
        // Log() only errors on invalid schema — not on network issues.
        // Network failures are retried internally and never surface as errors.
        log.Printf("invalid event schema: %v", err)
    }
}
```

**SDK internal design:**

```
client.Log(event)
    │
    ▼
Validate schema (synchronous, <1µs)
    │
    ▼
Append to ring buffer (thread-safe, lock-free)
    │
    ├── Buffer reaches 25 events → flush triggered
    └── 500ms ticker fires     → flush triggered
              │
              ▼
         HTTP POST /v1/events (batch of up to 25)
              │
         On failure: exponential backoff
              │   Attempt 1: 100ms delay
              │   Attempt 2: 200ms delay
              │   Attempt 3: 400ms delay
              └── After 3 failures: log warning to stderr, discard batch
```

**Published at:** `pkg.go.dev/github.com/YOUR_USERNAME/logknot-core/sdk/go` *(at v0.1.0)*

---

## Tech Stack

| Layer | Service / Tool | Purpose in LogKnot |
|---|---|---|
| **Language** | Go 1.22+ | All Lambda handlers and the Go SDK |
| **Compute** | AWS Lambda | Serverless function execution — scales to zero |
| **API** | AWS API Gateway (REST) | Routing, API key auth, usage plans, request throttling |
| **Primary storage** | AWS DynamoDB | Append-only event store (PITR + Streams enabled) |
| **Long-term archive** | AWS S3 + Glacier Instant Retrieval | 7-year HIPAA retention, auto-tiered by EventBridge |
| **Export storage** | AWS S3 | PDF and CSV export objects, pre-signed URLs (15-min TTL) |
| **Auth — SDK** | SHA-256 hashed API keys | Per-tenant auth stored as hash, never raw value |
| **Auth — Dashboard** | Amazon Cognito | Email + password, JWT tokens stored in memory only |
| **Alert pipeline** | AWS SNS + SES | Anomaly alerts fanned out to project owner via email |
| **Event pipeline** | DynamoDB Streams | NEW_IMAGE stream triggers anomaly Lambda and archival |
| **Scheduler** | Amazon EventBridge | Nightly usage metering and Glacier archival trigger |
| **WAF** | AWS WAF | SQLi, XSS, known-bad-bot rules on API Gateway |
| **CDN** | Amazon CloudFront + S3 | Dashboard static asset delivery from global edge |
| **Dashboard** | React + Vite | Frontend UI served from CloudFront, Cognito-authenticated |
| **Infrastructure as code** | AWS SAM | `template.yaml` defines all Lambda, DynamoDB, S3, IAM |
| **CI/CD** | GitHub Actions | `test → lint → deploy` pipeline, main-branch protection |
| **Billing** | Stripe | Checkout sessions, subscription webhooks, usage metering |
| **PDF generation** | gofpdf | SOC 2 compliance summary PDF rendered server-side |

---

## Repository Structure

```
logknot-core/
│
├── cmd/                           # Lambda function entry points (one per handler)
│   ├── ingest/
│   │   └── main.go                # POST /v1/events
│   ├── query/
│   │   └── main.go                # GET /v1/events  +  GET /v1/events/:id
│   ├── streams/
│   │   └── main.go                # DynamoDB Streams — anomaly detection
│   ├── export/
│   │   └── main.go                # POST /v1/exports (CSV + PDF)
│   ├── billing/
│   │   └── main.go                # Stripe webhook handler
│   ├── metering/
│   │   └── main.go                # Nightly usage counter (EventBridge)
│   └── keymanager/
│       └── main.go                # API key issuance and rotation
│
├── internal/                      # Shared packages — not part of public API
│   ├── dynamo/
│   │   ├── client.go              # DynamoDB client wrapper, connection pooling
│   │   └── client_test.go
│   ├── validator/
│   │   ├── event.go               # Event schema validation rules
│   │   └── event_test.go
│   ├── ulid/
│   │   ├── generate.go            # ULID generation (oklog/ulid)
│   │   └── generate_test.go
│   ├── auth/
│   │   ├── keys.go                # SHA-256 key hash and comparison
│   │   └── keys_test.go
│   └── anomaly/
│       ├── rules.go               # Anomaly detection rule engine
│       └── rules_test.go
│
├── sdk/
│   └── go/                        # Go SDK — published to pkg.go.dev
│       ├── client.go              # New(), Log(), Flush()
│       ├── client_test.go
│       ├── event.go               # Event, Actor, Resource types
│       ├── buffer.go              # Ring buffer, batch flush, retry
│       ├── buffer_test.go
│       └── README.md              # SDK-specific documentation
│
├── dashboard/                     # React + Vite frontend
│   ├── src/
│   │   ├── components/            # Reusable UI components
│   │   ├── pages/                 # Route-level page components
│   │   └── api/                   # API client — wraps /v1/* endpoints
│   ├── index.html
│   └── package.json
│
├── docs/
│   ├── API.md                     # Full REST API reference
│   ├── ARCHITECTURE.md            # Architecture decision records (ADRs)
│   ├── SECURITY.md                # Security model, threat model, disclosure policy
│   └── CONTRIBUTING.md            # Contribution guide
│
├── template.yaml                  # AWS SAM — all infrastructure declared here
├── samconfig.toml                 # SAM deploy targets: dev, staging, prod
│
├── .github/
│   └── workflows/
│       ├── test.yml               # go test ./... + golangci-lint on every push
│       ├── deploy-backend.yml     # sam build + sam deploy on merge to main
│       └── deploy-dashboard.yml  # npm run build + s3 sync + CF invalidation
│
├── dev.env.example                # Environment variable template for local dev
├── CONTRIBUTING.md
├── SECURITY.md
└── LICENSE
```

---

## Local Development

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Go | 1.22+ | [go.dev/dl](https://go.dev/dl/) |
| AWS CLI | v2+ | [aws.amazon.com/cli](https://aws.amazon.com/cli/) |
| AWS SAM CLI | latest | [docs.aws.amazon.com/sam](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) |
| Docker | latest | [docker.com](https://www.docker.com/) — required for DynamoDB Local and `sam local` |
| Node.js | 20+ | [nodejs.org](https://nodejs.org/) — dashboard only |

### Setup

```bash
# 1. Clone the repository
git clone https://github.com/YOUR_USERNAME/logknot-core.git
cd logknot-core

# 2. Copy the environment template
cp dev.env.example dev.env
# Edit dev.env — fill in your AWS account IDs, region, and test keys

# 3. Download Go module dependencies
go mod download

# 4. Run the full test suite
go test ./...

# 5. Run with race detector — required to pass before any PR
go test -race ./...

# 6. Run the linter
golangci-lint run ./...

# 7. Start DynamoDB Local in Docker
docker run -d -p 8000:8000 --name dynamodb-local amazon/dynamodb-local

# 8. Start the SAM local API (simulates API Gateway + Lambda)
sam local start-api --env-vars dev.env

# 9. (Optional) Start the dashboard
cd dashboard && npm install && npm run dev
```

### Environment variables

```bash
# DynamoDB table names (local)
EVENTS_TABLE_NAME=logknot-events-dev
KEYS_TABLE_NAME=logknot-keys-dev

# S3 bucket names (local or localstack)
EXPORTS_BUCKET=logknot-exports-dev

# AWS region
AWS_REGION=ap-south-1

# Stripe (use test mode keys)
STRIPE_WEBHOOK_SECRET=whsec_test_...

# SES sender (must be verified in SES sandbox)
SES_FROM_ADDRESS=alerts@logknot.dev
```

### Test the ingest endpoint

```bash
# With SAM local running on port 3000:
curl -X POST http://localhost:3000/v1/events \
  -H "x-api-key: test_key_dev" \
  -H "Content-Type: application/json" \
  -d '{
    "actor":    { "id": "user_001", "type": "user" },
    "action":   "document.delete",
    "resource": { "id": "doc_999", "type": "document" }
  }'

# Expected HTTP 201:
# {"eventId":"01J5K2M3N4P5Q6R7S8T9U0V1W2","status":"created"}

# Verify the event was written (local DynamoDB):
aws dynamodb scan \
  --table-name logknot-events-dev \
  --endpoint-url http://localhost:8000
```

### Test the IAM deny policy

```bash
# This should return AccessDeniedException — confirming immutability enforcement
aws dynamodb update-item \
  --table-name logknot-events-prod \
  --key '{"projectId":{"S":"proj_001"},"eventId":{"S":"01J5K..."}}' \
  --update-expression "SET #a = :v" \
  --expression-attribute-names '{"#a":"action"}' \
  --expression-attribute-values '{":v":{"S":"tampered"}}' \
  --region ap-south-1

# Expected: An error occurred (AccessDeniedException)
```

---

## Roadmap

> **Development philosophy:** Each phase is implemented and validated end-to-end before
> the next begins. No phase is considered complete until its test suite passes with
> the race detector enabled.

```
Phase 0  ████████████████████  COMPLETE
Phase 1  ░░░░░░░░░░░░░░░░░░░░  In design
Phase 2  ░░░░░░░░░░░░░░░░░░░░  Planned
Phase 3  ░░░░░░░░░░░░░░░░░░░░  Planned
Phase 4  ░░░░░░░░░░░░░░░░░░░░  Planned
```

### Phase 0 — Engineering Foundation *(Complete)*

- [x] System architecture design (5-layer serverless model)
- [x] DynamoDB schema design (PK/SK/GSI/LSI layout)
- [x] IAM deny policy design and documentation
- [x] SDK interface specification frozen
- [x] Repository structure established
- [x] `logknot-landing` deployed at [logknot.com](https://logknot.com)

### Phase 1 — Core Ingestion API

- [ ] `POST /v1/events` — Ingest Lambda (Go)
  - [ ] EventSchema struct with `go-playground/validator`
  - [ ] ULID generation via `oklog/ulid`
  - [ ] `dynamodb.BatchWriteItem` (up to 25 events/call)
  - [ ] Returns `{ eventId, status }` on HTTP 201
- [ ] IAM deny policy applied to all Lambda execution roles
- [ ] Automated IAM policy enforcement test (expect `AccessDeniedException`)
- [ ] API Gateway usage plans: Free (50K/mo), Starter (1M/mo), Growth (10M/mo)
- [ ] SHA-256 API key hashing in Key Manager Lambda
- [ ] `GET /v1/events` — Query Lambda (Go)
  - [ ] FilterExpression on actorId, action, resourceType, date range
  - [ ] Cursor pagination via `LastEvaluatedKey`
  - [ ] Response: `{ events: [...], nextCursor? }`
- [ ] `GET /v1/events/:id` — Single event retrieval
- [ ] Rate limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`
- [ ] `X-Request-ID` header on all responses
- [ ] CloudWatch structured JSON logging (requestId, projectId, duration, statusCode)
- [ ] Table-driven unit tests for all Lambda handlers
- [ ] Integration test: POST event → query → assert ULID in response

### Phase 2 — Streams, Export and Anomaly Detection

- [ ] DynamoDB Streams enabled (NEW_IMAGE view type)
- [ ] Streams Lambda (Go)
  - [ ] Rule 1: bulk delete detection (>50 DELETE/60s from one actor)
  - [ ] Rule 2: privilege escalation (action contains "admin" / "role_change")
  - [ ] Rule 3: off-hours access (outside 07:00–22:00 in project timezone)
  - [ ] Unit tests for all 3 rules with mock stream records
- [ ] SNS → SES anomaly email pipeline
- [ ] SQS Dead Letter Queue for failed stream records
- [ ] `POST /v1/exports` — Export Lambda (Go)
  - [ ] CSV generation via `encoding/csv`
  - [ ] SOC 2 PDF generation via `gofpdf`
  - [ ] Upload to S3, return pre-signed URL (15-min TTL)
- [ ] Nightly S3 Glacier archive Lambda (EventBridge trigger)

### Phase 3 — SDK, Dashboard and Billing

- [ ] Go SDK (`sdk/go/`)
  - [ ] `New(apiKey string) *Client`
  - [ ] `Log(event Event) error` — non-blocking, buffer append
  - [ ] `Flush() error` — drain buffer, wait for in-flight requests
  - [ ] Ring buffer: flush on 25 events OR 500ms ticker
  - [ ] Retry: exponential backoff (100ms / 200ms / 400ms, 3 attempts)
  - [ ] `go test -race` — zero data races
  - [ ] Published to `pkg.go.dev` at `v0.1.0`
- [ ] React + Vite dashboard
  - [ ] Cognito Hosted UI login — JWT stored in memory only
  - [ ] Project selector dropdown
  - [ ] Event table (actor, action, resource, timestamp) — paginated
  - [ ] Filter bar (actorId, action, date range) — 300ms debounce
  - [ ] Export button → POST /v1/exports → download via pre-signed URL
  - [ ] Deployed to S3 + CloudFront
- [ ] Stripe Checkout sessions wired to API key activation
- [ ] Stripe webhook Lambda (idempotent via processed-event DynamoDB table)
- [ ] Nightly usage metering Lambda

### Phase 4 — Hardening and Public Launch

- [ ] k6 load tests
  - [ ] Ramp-up: 0 → 100 VU over 2 minutes — target p95 < 300ms, zero errors
  - [ ] Spike: 200 VU instant — verify Lambda scales, throttles return 429
  - [ ] Soak: 50 VU for 5 minutes — verify no memory leak or cold-start degradation
- [ ] OWASP API Security Top 10 checklist
- [ ] AWS WAF rules enabled (SQLi, XSS, managed bot rules)
- [ ] Lambda reserved concurrency set (200 per function for cost control)
- [ ] GitHub Actions pipeline
  - [ ] `test.yml`: `go test ./...` + `go test -race ./...` + `golangci-lint`
  - [ ] `deploy-backend.yml`: `sam build` + `sam deploy --no-confirm` on merge to main
  - [ ] `deploy-dashboard.yml`: `npm run build` + `s3 sync` + CF invalidation
  - [ ] Branch protection: tests must pass before merge to main
- [ ] Public beta launch

---

## Contributing

Contributions, architecture feedback, and security reviews are welcome.

**Before contributing:**

1. Read [CONTRIBUTING.md](./CONTRIBUTING.md)
2. Open an issue before starting significant work — align on approach first
3. All PRs must target a feature branch, not `main` directly

**Quality gates — all must pass before review:**

```bash
go test ./...            # Zero test failures
go test -race ./...      # Zero race conditions detected
golangci-lint run ./...  # Zero lint errors
```

**Hard rules — no exceptions:**

- No `dynamodb:UpdateItem` or `dynamodb:DeleteItem` calls on any table prefixed
  `logknot-events-*` under any circumstances, in any code path
- No `fmt.Println` in Lambda handlers — all logs must be structured JSON to stdout
- No hardcoded configuration values — all environment-specific values via `dev.env`
- Table-driven tests required for all validation and business logic
- New Lambda handlers must include an integration test

**Branch naming:**

```
feat/short-description        # New functionality
fix/short-description         # Bug fixes
refactor/short-description    # Internal restructuring, no behaviour change
docs/short-description        # Documentation only
test/short-description        # Test coverage improvements
```

---

## Security

**Reporting vulnerabilities:**
Do **not** open a public GitHub issue for security matters.
Report privately to `security@logknot.dev`.
Critical issues receive a response within 48 hours.

See [SECURITY.md](./SECURITY.md) and [/.well-known/security.txt](https://logknot.com/.well-known/security.txt).

**Security model:**

| Control | Mechanism | Scope |
|---|---|---|
| Event immutability | IAM `Deny` on `UpdateItem` + `DeleteItem` | All Lambda execution roles |
| API authentication | SHA-256 hashed keys, never stored raw | API Gateway — per-request |
| Dashboard auth | Amazon Cognito JWT, memory-only storage | React dashboard |
| Data in transit | TLS 1.2+ enforced by API Gateway | All endpoints |
| Data at rest | DynamoDB SSE-S3, S3 SSE-S3 | Events, keys, exports |
| Network layer | AWS WAF — SQLi, XSS, bot rules | API Gateway |
| Webhook delivery | HMAC-SHA256 signed payloads | Outbound webhooks |
| PII policy | Actor IDs and resource IDs only — no personal data stored | All storage |
| Stripe webhooks | `Stripe-Signature` header verified before processing | Billing Lambda |
| Concurrency safety | `go test -race` required in CI | Go SDK and shared packages |

---

## Related Repository

| Repository | Audience | Contents |
|---|---|---|
| [`logknot-landing`](https://github.com/YOUR_USERNAME/logknot-landing) | Users, investors | Marketing site — HTML/CSS/JS, Vercel, Tally.so waitlist |
| [`logknot-core`](https://github.com/YOUR_USERNAME/logknot-core) | ← You are here | Backend engine — Go, AWS SAM, all infrastructure |

---

## License

MIT — see [LICENSE](./LICENSE) for full terms.

---

<div align="center">

**logknot-core** · the backend engine for [logknot.com](https://logknot.com)

*Pre-launch · Built by a solo founder · Shaped by the engineering community*

</div>
