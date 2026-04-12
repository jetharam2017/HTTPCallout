# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Salesforce DX project** using the **Org Development Model** (non-source-tracked org). It implements an **HTTPCallout Framework** — a reusable Apex framework for making synchronous and asynchronous HTTP callouts from Salesforce, driven by Custom Metadata (`HTTPCalloutConfiguration__mdt`).

The project also tracks CPQ (`SBQQ__`) and Billing (`blng__`) managed package metadata via the manifest.

## Key Commands

### Salesforce CLI (Metadata)
```bash
# Retrieve metadata from connected org using manifest
sf project retrieve start --manifest manifest/package.xml

# Deploy metadata to connected org using manifest
sf project deploy start --manifest manifest/package.xml

# List connected orgs
sf org list

# Open default org in browser
sf org open
```

### Linting & Formatting
```bash
npm run lint            # ESLint on LWC components
npm run prettier        # Auto-format all supported file types
npm run prettier:verify # Check formatting without writing
```

### Apex Tests (run from org, not locally)
```bash
sf apex run test --class-names HTTPCalloutServiceTest --result-format human
sf apex run test --class-names HTTPCalloutAsyncServiceTest --result-format human
```

### LWC Unit Tests
```bash
npm run test:unit           # Run once
npm run test:unit:watch     # Watch mode
npm run test:unit:coverage  # With coverage
```

## Architecture

### HTTPCallout Framework (Apex)

The framework lives in `force-app/main/default/classes/` and has three main classes:

- **`HTTPCalloutService`** — Synchronous callouts. Can be instantiated with or without a Custom Metadata name. When given a metadata name, it auto-loads endpoint, method, headers, URL params, body, timeout, and certificate from `HTTPCalloutConfiguration__mdt`. Call `sendRequest()` to execute or `getRequest()` to inspect the built `HTTPRequest` without sending.

- **`HTTPCalloutAsyncService`** — Asynchronous callouts using the Salesforce Continuation API. Accepts up to `CONTINUATION_LIMIT` (3) concurrent requests. Can be instantiated with raw `HTTPRequest` objects or Custom Metadata names. Call `sendRequest(responseMethodName)` to get a `Continuation` object, then `getResponse(requestLabels)` in the callback method.

- **`HTTPCalloutFrameworkException`** — Custom exception class holding all error message constants used across the framework.

### Custom Metadata (`HTTPCalloutConfiguration__mdt`)

Stores callout configurations. Key fields: `Endpoint__c`, `Method__c`, `Body__c`, `Timeout__c`, `IsCompressed__c`, `CertificateName__c`, `HeaderParameters__c`, `URLParameters__c`.

`HeaderParameters__c` and `URLParameters__c` are stored as newline-separated `key:value` pairs parsed at runtime by `HTTPCalloutService`.

### Test Infrastructure

- **`HTTPCalloutServiceMock`** — Single-endpoint mock implementing `HttpCalloutMock`.
- **`HTTPCalloutServiceMultiMock`** — Multi-endpoint mock that routes by URL, used in tests needing multiple distinct callout responses.

### Manifest (`manifest/package.xml`)

Organized into sections:
1. **Existing project components** — The HTTPCallout framework classes, custom metadata records, layout, object, and remote site setting.
2. **CPQ custom fields on standard objects** — Fields prefixed `SBQQ__` on `SBQQ__Quote`, `Opportunity`, `Account`, `Product2`, `Order`, `OrderItem`, `Contract`.
3. **CPQ/Billing managed objects and metadata** — `SBQQ__*` and `blng__*` objects, layouts, flows, permission sets, etc.

> **Note:** `SBQQ__` fields on the CPQ Quote object use `SBQQ__Quote.SBQQ__FieldName__c` (not `Quote.SBQQ__FieldName__c`), since `SBQQ__Quote__c` is a managed custom object, not the standard Quote object.

## Development Model Notes

- This project targets a **non-source-tracked org** (`SalesforceCast`, alias for `salesforcecasts@dnb.com`). Use `retrieve`/`deploy`, **not** `push`/`pull`.
- `sourceApiVersion` is set to `48.0` in `sfdx-project.json`.
- **Never deploy directly to production** from the CLI — use packaging or Metadata API with test execution.
