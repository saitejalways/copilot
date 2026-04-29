# Infrastructure - AWS CDK Configuration

## Overview

This directory contains AWS CDK (Cloud Development Kit) infrastructure-as-code for the Anti-Money Laundering (AML) API deployment.

## Build Configuration (`bin/build.ts`)

The `build.ts` file establishes AWS configurations that will be deployed across different environments (DEV, RC, STG, UAT, PROD).

### Key Configurations Managed

#### Application Properties

- **App Name**: `anti-money-laundering`
- **App ID**: `aml`
- **Base Path**: `/api/aml`
- **Container Port**: `8080`
- **ALB Priority**: `42`
- **ECR Name**: `alpha/anti-money-laundering-api`

#### Event Bus Filter Policy (EventBridge Subscriptions)

The `eventBusFilterPolicy` defines which events from the central event bus this API will receive:

**Domains subscribed to:**

- `unit-of-work`
- `data-migration`
- `client`
- `user`
- `document-job`
- `document-template`
- `anduin`
- `entity-attributes`
- `screening-summarized-result`

**Modules subscribed to:**

- `work-connect`
- `thirdpartyrisk`
- `platform`
- `document-assembly`
- `deliverable-generator`

#### KMS Key Access

Environment-specific KMS key aliases for decrypting data from legacy systems:

- **DEV**: `alpha-app-app-user-data-key`
- **RC**: `platform-uat-key`, `alpha-app-app-user-data-key`
- **STG**: `kms-platform-stg`, `kms-platform-prod`
- **UAT**: `kms-platform-stg2`, `kms-platform-prod`
- **PROD**: `kms-platform-prod`

#### S3 Bucket Access

The API has read/write access to:

- **Documents bucket**: `aca-{env}-alpha-aml-documents-{region}`
- **Short-term bucket**: `aca-{env}-alpha-short-term-{region}`

Allowed S3 actions:

- `s3:ListBucket`
- `s3:PutObject`
- `s3:DeleteObject`
- `s3:DeleteObjectVersion`
- `s3:GetObject`

## Event Publishing and Listening

For detailed information on event publishing, event structure, and event definitions, see [Source/AntiMoneyLaundering/.github/copilot-instructions.md](../../Source/AntiMoneyLaundering/.github/copilot-instructions.md#event-publishing-and-listening).

This infrastructure documentation focuses on the infrastructure deployment workflow required when adding event listeners.

### Infrastructure Deployment Workflow

**IMPORTANT**: Changes to the event subscription filter require infrastructure deployment.

1. **Include infrastructure changes in your PR**: Update `infrastructure/bin/build.ts` with the new subscription filter
2. **After PR is merged** (NOT before):
   - Run the `infrastructure` deploy GitHub workflow against the branch your PR was merged into
3. **When the Release Monitor ticket is created in Jira**:
   - Add a note that **the infrastructure project must be deployed** as part of this release
4. **After deployment**:
   - Verify the subscription filter policy is updated in AWS EventBridge console

**Deployment Verification:**

- Check AWS EventBridge console for the AML API's event rule
- Verify the filter policy includes your new module/domain
- Test that events are received by the listener

## Deployment

This infrastructure is deployed using AWS CDK. Deployments are managed through GitHub workflows.

### Deployment Command

```bash
# Deploy to specific region
npm run deploy -- {region}
```

### Deployment Regions

The AML API is deployed to multiple AWS regions to support global availability.
