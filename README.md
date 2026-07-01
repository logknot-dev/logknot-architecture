<div align="center">

```
 _                _  __          _
| |    ___   __ _| |/ /_ __  ___| |_
| |   / _ \ / _` | ' /| '_ \/ _ \ __|
| |__| (_) | (_| | . \| | | | (_) | |_
|_____\___/ \__, |_|\_\_| |_|\___/ \__|
             |___/
```

# LogKnot Architecture

**Tamper-evident audit trail architecture for security-conscious SaaS teams**

[![Status](https://img.shields.io/badge/Status-Private%20Beta%20Build-f59e0b?style=flat-square)](https://www.logknot.com/)
[![Backend](https://img.shields.io/badge/Backend-Go%20%2B%20AWS%20Serverless-00ADD8?style=flat-square&logo=go&logoColor=white)](https://go.dev/)
[![Architecture](https://img.shields.io/badge/Architecture-Public%20Review-22c55e?style=flat-square)](https://github.com/logknot-dev/logknot-architecture)

**Website:** [logknot.com](https://www.logknot.com/)  
**Design Partner Waitlist:** [Join Private Beta](https://tally.so/r/kdRrJJ)  
**Built by:** LogKnot Technologies

</div>

---

## Purpose of This Repository

This repository documents the public backend architecture and security direction for LogKnot.
It is intended for engineering teams, security reviewers, and early design partners who want to understand how LogKnot is being built before joining Private Beta.

This repository intentionally shares the **architecture principles and trust model**, not private implementation details, customer-specific configurations, secrets, or operational runbooks.

---

## Private Beta Status

LogKnot is currently in **Private Beta development**.

### Available / Implemented Foundation

- Immutable audit event ingestion foundation
- Hash-chain based tamper-evidence
- Merkle receipt generation foundation
- AWS Serverless backend using Go
- JWT authentication middleware
- Tenant context enforcement
- Role-based access control foundation
- Cognito identity infrastructure
- Cognito JWT validation using RS256 / JWKS
- Country-based region policy for early design partner markets
- DynamoDB write-safety and WORM-oriented controls
- CI checks for build, test, vet, lint, and SAM validation

### Planned / Private Beta Roadmap

- Tenant signup and onboarding flow
- Tenant metadata repository
- End-to-end integration testing
- Load testing and performance baseline
- Developer documentation and Postman examples
- Go SDK refresh aligned with Cognito/JWT auth
- Node.js SDK
- Compliance evidence exports
- SOC 2 style report formatting
- AI-assisted anomaly analysis
- Paddle billing integration

Planned items are not marketed as production-ready capabilities until they are implemented, tested, and included in Private Beta scope.

---

## Vision

To help organizations worldwide build trust through verifiable, tamper-evident audit trails that strengthen security, simplify compliance, and support sustainable business growth.

We believe every organization—regardless of size—should have access to trustworthy audit records that help demonstrate accountability, meet compliance requirements, and earn the confidence of customers, partners, and regulators.

---

## Core Backend Workflow

LogKnot is designed around one security principle:

> Audit records should be verifiable after they are written, and difficult to alter without detection.

High-level request flow:

```text
Client Application
    ↓
HTTPS API Request
    ↓
API Gateway
    ↓
JWT Authentication
    ↓
Tenant Context Extraction
    ↓
RBAC Authorization
    ↓
Input Validation
    ↓
Immutable Event Processing
    ↓
DynamoDB Append-Only Storage
    ↓
Hash Chain / Merkle Integrity Layer
    ↓
Receipt / Verification Response
```

This workflow separates identity, authorization, tenant isolation, storage safety, and integrity verification into clear layers.

---

## Security Model

LogKnot follows a defense-in-depth model.

| Layer | Responsibility |
|---|---|
| Identity | Cognito-backed JWT validation with RS256 and JWKS verification |
| Tenant boundary | Tenant and organization context comes from validated claims, not request bodies |
| Authorization | RBAC middleware protects sensitive API operations |
| Region policy | Tenant country maps to an approved AWS region during onboarding |
| Write safety | Event records are append-oriented and protected from unsafe overwrite paths |
| Integrity | Hash chain and Merkle structures support tamper-evidence |
| Infrastructure | AWS IAM controls restrict destructive write behavior where required |
| Observability | CI and runtime controls support safer operations before Private Beta |

---

## Tenant and Region Isolation

For Private Beta, LogKnot is being designed for early design partners across:

- India
- Singapore
- Malaysia
- UAE
- Qatar
- Philippines

Region assignment is server-controlled. Clients do not choose or override their region after onboarding.

```text
Tenant Country
    ↓
Region Policy
    ↓
Tenant Metadata
    ↓
Request-Time Validation
    ↓
Region-Aware Storage Access
```

This provides a clean foundation for future data-residency controls while keeping Private Beta operationally manageable.

---

## Immutability and Tamper Evidence

LogKnot does not rely on a single control to protect audit history.

The backend combines:

- append-oriented event creation
- conditional write safety
- IAM-level restrictions for destructive operations
- deterministic event hashing
- hash-chain continuity
- Merkle receipt generation foundation
- independent infrastructure auditability through AWS-native controls

The goal is not to claim that no system can ever be compromised. The goal is to make unauthorized modification difficult, visible, and independently reviewable.

---

## Backend Components

```text
API Gateway
    ↓
Go Lambda Runtime
    ↓
Middleware Layer
    ├── Authentication
    ├── Claims Context
    ├── RBAC Authorization
    └── Tenant Enforcement
    ↓
Business Logic Layer
    ├── Event Ingestion
    ├── Hash Chain
    ├── Merkle Receipt Foundation
    └── Verification Foundation
    ↓
Repository Layer
    ├── DynamoDB Event Store
    ├── Tenant Metadata Store
    └── Future Billing / Usage Metadata
    ↓
AWS Infrastructure
    ├── DynamoDB
    ├── Cognito
    ├── IAM
    ├── Lambda
    └── API Gateway
```

The public repository describes the architecture. The private backend repository contains implementation details, deployment configuration, and internal engineering history.

---

## Technology Stack

| Area | Technology |
|---|---|
| Backend language | Go |
| Runtime | AWS Lambda custom runtime on Amazon Linux 2023 |
| API layer | Amazon API Gateway |
| Identity | Amazon Cognito |
| Token validation | RS256 JWT validation using JWKS |
| Primary storage | Amazon DynamoDB |
| Infrastructure as Code | AWS SAM |
| CI quality gates | Go test, go vet, golangci-lint, SAM validate/build |
| Integrity model | Hash chain and Merkle receipt foundation |
| Billing | Paddle planned |

---

## API Design Direction

Initial Private Beta API scope focuses on a small, reliable core:

```text
POST /v1/events        Ingest audit event
GET  /v1/events        Query audit events
GET  /v1/events/{id}   Fetch one audit event
POST /v1/verify        Verify event integrity / receipt
```

API contracts will be documented before design partner testing. SDKs are planned after the API contract stabilizes.

---

## What We Intentionally Do Not Publish

To protect customers, infrastructure, and product execution, this public architecture repository does not publish:

- AWS account identifiers
- private table or bucket names
- secrets or environment values
- customer-specific architecture
- internal deployment runbooks
- unreleased commercial pricing experiments
- operational incident procedures
- private implementation branches

This keeps the architecture transparent while protecting operational security.

---

## Design Partner Evaluation

LogKnot is currently looking for early design partner feedback from engineering teams building:

- SaaS platforms
- HRTech systems
- FinTech platforms
- ERP products
- Healthcare software
- Cybersecurity tools

Ideal design partners care about:

- trustworthy audit history
- tenant isolation
- compliance evidence
- low-operational-overhead audit logging
- verifiable event integrity
- developer-friendly API integration

Join the waitlist: [https://tally.so/r/kdRrJJ](https://tally.so/r/kdRrJJ)

---

## Engineering Principles

LogKnot development follows these principles:

- security decisions should be explicit
- tenant isolation must be enforced server-side
- audit records must not depend on application trust alone
- write paths should be safe by default
- infrastructure should be reproducible
- tests should protect security regressions
- marketing claims must match implemented capability

---

## Related Links

| Link | Purpose |
|---|---|
| [LogKnot Website](https://www.logknot.com/) | Product overview and Private Beta positioning |
| [Design Partner Waitlist](https://tally.so/r/kdRrJJ) | Private Beta interest form |
| [Architecture Repository](https://github.com/logknot-dev/logknot-architecture) | Public backend architecture and trust model |

---

## Security Contact

Security feedback can be shared privately through:

```text
security@logknot.com
```

Please do not open public GitHub issues for suspected vulnerabilities.

---

<div align="center">

**LogKnot Technologies**  
*Trustworthy audit trails for security and compliance-conscious teams.*

</div>
