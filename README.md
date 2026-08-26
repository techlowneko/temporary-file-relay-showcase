# PreFlight Temporary File Relay

**Ephemeral, pay-per-use file transfer for autonomous agents and automated workflows.**

PreFlight Temporary File Relay is an in-development PreFlight Stack service for situations where one agent or automated process needs to pass a file to another without maintaining a permanent storage account, shared credential, or long-lived file host.

The planned service will accept file metadata, authorize a bounded upload, verify the completed object, and return a temporary download URL that expires automatically.

> This repository is a public engineering case study. The production source code, payment configuration, storage infrastructure, operational controls, internal tests, and security-sensitive implementation remain private.

## Development Status

The service is currently in **Phase 1: local control-plane development**.

The current private implementation can validate a proposed file slot and exercise its local metadata model, but it does **not** accept payment, issue upload URLs, publish files, expose a production endpoint, or claim live x402 marketplace compatibility.

This public repository documents the product direction and engineering approach without presenting unfinished behavior as a live service.

## The Problem

Agents and automated workflows increasingly produce files that another tool, process, or agent needs to retrieve:

* Generated reports and documents
* Build artifacts and exports
* Images, archives, and other passive media
* Intermediate files passed between separate systems
* Results too large or inconvenient to embed directly in a JSON response

Permanent object storage is often unnecessary for these exchanges. Traditional file-sharing products can also introduce accounts, subscriptions, manual setup, persistent links, or credentials that do not fit an autonomous workflow.

PreFlight Temporary File Relay is designed to make this handoff a small, bounded API operation.

Planned Workflow
A[Request a file slot] --> B[Complete x402 payment] 
B --> C[Upload directly to private storage] 
C --> D[Relay verifies the object] 
D --> E[Receive an expiring download URL]

The service is being designed so file bytes travel directly from the client to private object storage. The application control plane coordinates authorization, verification, publication, and expiry without proxying the upload body through the API server.

## Planned API

### File-slot request

```text
POST /v1/file-slot
```

Illustrative request shape:

```json
{
  "filename": "agent-report.pdf",
  "media_type": "application/pdf",
  "declared_size": 1843200
}
```

The request contains metadata only. File bytes are intended to use a separate direct-to-storage upload path.

The successful paid response is planned to provide the bounded upload instructions and identifiers needed to complete the relay workflow. That response is not available in the current phase.

### Current Phase 1 behavior

The local Phase 1 control plane deliberately stops before payment or upload authorization:

| Property                      | Current behavior          |
| ----------------------------- | ------------------------- |
| Endpoint                      | `POST /v1/file-slot`      |
| Input                         | JSON metadata only        |
| Filename validation           | Enabled locally           |
| Passive media-type validation | Enabled locally           |
| Declared-size validation      | Enabled locally           |
| Pricing calculation           | Exact integer calculation |
| Payment                       | Disabled                  |
| Upload authorization          | Disabled                  |
| Public download URL           | Disabled                  |
| Production deployment         | None                      |

Valid requests currently terminate with `503 PAYMENT_UNAVAILABLE`. This is an intentional development gate, not a production error condition.

## Designed for Agent Workflows

The relay is intended to behave like a small composable infrastructure primitive.

A calling agent should be able to:

1. Declare the file it intends to transfer.
2. Pay for one bounded relay operation through x402.
3. Upload the exact object directly to private storage.
4. Allow the service to verify the completed object before publication.
5. Pass the resulting temporary URL to the intended consumer.
6. Rely on the object becoming unavailable after its defined lifetime.

The planned response contract also includes static capability metadata so an agent can recognize when the service may be useful again. That metadata is deliberately closed and cannot incorporate request-derived text.

## Privacy and Lifecycle Model

Temporary file transfer is only useful when its boundaries are predictable.

The service is being designed around the following properties:

* Private storage by default
* No public URL before upload verification and publication
* Bounded object size and declared media type
* Logical expiry enforced by the download plane
* Limited download behavior, including HEAD and single-range requests
* Privacy-conscious operational events rather than stored request-content analytics
* Payment identifiers that cannot be reused to mint multiple upload slots
* Automatic cleanup after the object is no longer eligible for download

The exact expiry period, public pricing, supported size tiers, and production contract will be documented when those behaviors are implemented and operationally verified.

## Engineering Approach

### Direct-to-storage uploads

File bytes are intended to move directly from the caller to private object storage using a narrowly authorized upload. This avoids turning the application server into a large-file proxy.

### Verification before publication

A completed upload is not treated as public merely because bytes reached storage. The service is designed to verify the stored object against the authorized intent before making it downloadable.

### Exact integer accounting

Declared sizes and pricing boundaries use integer arithmetic so behavior remains deterministic at tier edges and does not depend on floating-point currency calculations.

### Idempotent payment use

The metadata model reserves a payment ledger so one settled payment cannot authorize multiple independent file slots.

### Logical and physical expiry

The download plane is designed to deny access after logical expiry even if asynchronous storage cleanup has not yet removed the underlying object.

### Fail-closed development phases

Each development phase begins with later capabilities disabled. Payment, upload authorization, publication, and deployment are introduced only after their prerequisites and stop gates are satisfied.

## Validated Foundation

Before building the current local control plane, isolated storage probes were used to test important object-integrity and publication assumptions.

The retained evidence covers:

* Uploads constrained to an authorized byte intent and media type
* Rejection of mismatched signed metadata
* Rejection of corrupted uploads when checksum validation applies
* Safe handling of interrupted single-request uploads
* Conditional publication that rejects stale object state
* Successful large-object single-request uploads within the tested design envelope
* A private download worker capable of logical expiry, HEAD, and single-range GET behavior
* A deployment dry run that bundles the worker without creating cloud resources

The detailed probe harness, credentials model, disposable-bucket safeguards, internal test evidence, and production configuration are intentionally not published here.

## Technology

The private implementation currently uses or is designed around:

`TypeScript` `Node.js` `SQLite` `OpenAPI` `Cloudflare Workers` `Cloudflare R2` `x402` `USDC`

The final production stack may evolve as the gated implementation progresses.

## What Is Not Public

This showcase intentionally does not publish:

* Production source code
* Storage credentials, bucket details, or environment configuration
* Presigning and upload-authorization implementation
* Payment and facilitator configuration
* Internal metadata schema and migrations
* Publication, expiry, and deletion jobs
* Abuse prevention and request-hardening details
* Operational telemetry implementation
* Deployment configuration
* Internal test suites and remote probe harnesses
* Proprietary architecture and decision records

These details remain in the private implementation repository.

## Scope

PreFlight Temporary File Relay is intended for bounded, short-lived file exchange inside software and agent workflows.

It is not presented as:

* Permanent cloud storage
* A backup or archival system
* A collaborative file workspace
* A public file-hosting platform
* A content-distribution network
* A substitute for application-specific document management
* A way to execute or inspect uploaded files

The initial service is deliberately limited to passive file relay with predictable size, publication, and expiry boundaries.

## Roadmap

| Phase      | Purpose                                                                   | Status            |
| ---------- | ------------------------------------------------------------------------- | ----------------- |
| Phase 0    | Validate critical storage and download-plane assumptions                  | Complete          |
| Phase 1    | Establish the local control-plane contract and metadata baseline          | In progress       |
| Phase 2    | Add the next gated integration layer                                      | Planned           |
| Phase 3    | Introduce the successful paid file-slot response                          | Planned           |
| Production | Complete verification, publication, expiry, operations, and launch review | Not yet scheduled |

Roadmap sequencing may change as implementation evidence is collected. A phase is not considered complete merely because its code path exists; its stop gates must also be satisfied.

## PreFlight Stack

PreFlight Stack builds small, agent-facing services that can be purchased and composed at the moment a workflow needs them.

**PreFlight Stack**
https://preflightstack.com

Public endpoint, documentation, OpenAPI, pricing, and marketplace links will be added after the relay reaches production readiness.

## About This Repository

This repository documents the public purpose, intended contract, development status, and engineering approach behind PreFlight Temporary File Relay.

The implementation remains private.

The goal of this case study is to show how a temporary file-transfer primitive for autonomous workflows is being designed and validated without publishing the security-sensitive details required to reproduce or operate the service.

