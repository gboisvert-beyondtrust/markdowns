# BeyondTrust Forms Backend - Technical Documentation

## Executive Summary

This document provides comprehensive technical details of the BeyondTrust Forms Backend system, designed to support the evolution to a YAML-based form system. This serverless backend manages multi-step trial form submissions with sophisticated fraud prevention, validation, enrichment, and provisioning capabilities.

**Version:** 1.0.0
**Node Version:** >=22
**Runtime:** AWS Lambda with Node.js 22.x
**Infrastructure:** AWS SAM (Serverless Application Model)

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [API Endpoints](#api-endpoints)
3. [Data Flow](#data-flow)
4. [Database Schema](#database-schema)
5. [Validation & Business Rules](#validation--business-rules)
6. [External Integrations](#external-integrations)
7. [Security & Fraud Prevention](#security--fraud-prevention)
8. [Form Processing Logic](#form-processing-logic)
9. [Environment Configuration](#environment-configuration)
10. [Development Setup](#development-setup)

---

## Architecture Overview

### System Architecture

The system follows a serverless event-driven architecture:

```
┌─────────────┐
│   Client    │
│  (Browser)  │
└──────┬──────┘
       │
       │ HTTPS
       ▼
┌─────────────────────┐
│  API Gateway        │
│  - /bootstrap       │
│  - /trial-submit    │
│  - /process         │
└──────┬──────────────┘
       │
       ▼
┌──────────────────────────────────────────────────┐
│              Lambda Functions                     │
│                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────┐│
│  │  Bootstrap   │  │Trial Handler │  │Process ││
│  │   Function   │  │   Function   │  │Function││
│  └──────┬───────┘  └──────┬───────┘  └───┬────┘│
│         │                  │               │     │
└─────────┼──────────────────┼───────────────┼─────┘
          │                  │               │
          │                  ▼               │
          │         ┌────────────────┐       │
          │         │   SQS Queue    │◄──────┘
          │         │  TrialQueue    │
          │         └────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────┐
│              Storage Layer                        │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌────────────────┐│
│  │ DynamoDB │  │    S3    │  │ Secrets Manager││
│  │  Tables  │  │  Bucket  │  │                ││
│  └──────────┘  └──────────┘  └────────────────┘│
└──────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────┐
│           External Services                       │
│                                                   │
│  • Cloudflare Turnstile (CAPTCHA)               │
│  • TeleSign (Phone Verification)                │
│  • 6Sense (Enrichment)                          │
│  • Clearbit (Enrichment)                        │
│  • Eloqua (Marketing Automation)                │
│  • Builder API (Trial Provisioning)             │
│  • MaxMind GeoIP2 (Geolocation)                 │
└──────────────────────────────────────────────────┘
```

### Core Components

#### 1. Lambda Functions

**Bootstrap Function** ([src/functions/bootstrap/index.ts](src/functions/bootstrap/index.ts))
- **Purpose:** Initialize form session and provide geolocation data
- **Trigger:** API Gateway GET /bootstrap
- **Key Features:**
  - MaxMind GeoIP2 database initialization (cached across invocations)
  - IP-based geolocation lookup
  - Referer security validation
  - Email parameter decryption (PHP-compatible)
  - Campaign ID extraction from query params

**Trial Handler Function** ([src/functions/trial-handler/index.ts](src/functions/trial-handler/index.ts))
- **Purpose:** Handle multi-step form submission process
- **Trigger:** API Gateway POST /trial-submit
- **Supported Steps:**
  - Step 1: Initial submission with email validation
  - Step 2: Phone verification via SMS/call
  - Step 3: Final submission and trial provisioning
- **Key Features:**
  - Zod-based schema validation
  - Turnstile CAPTCHA validation
  - Rate limiting (email and phone)
  - Email domain checking (free email detection)
  - Country blocking/allowlisting
  - Company enrichment lookup
  - Submission deduplication
  - Audit trail tracking

**Trial Processor Function** ([src/functions/trial-processor/index.ts](src/functions/trial-processor/index.ts))
- **Purpose:** Process submissions asynchronously and provision trials
- **Triggers:**
  - SQS Queue (production)
  - API Gateway POST /process (local testing)
- **Key Features:**
  - SQS message deduplication
  - Eloqua integration with enrichment
  - Builder API provisioning
  - Spam scoring
  - Submission filtering based on flags (green/yellow/red)

---

## API Endpoints

### 1. Bootstrap Endpoint

**Endpoint:** `GET /bootstrap`

**Purpose:** Initialize form session and provide client-specific data

**Query Parameters:**
- `campid` (optional): Salesforce Campaign ID
- `em` (optional): Encrypted email address (PHP-encrypted)

**Response:**
```json
{
  "message": "OK",
  "campid": "7012S0000004QbuQAE",
  "em": "user@example.com",
  "geolocation": {
    "userIp": "192.0.2.1",
    "geoUserCountryIsoCode": "US",
    "geoUserCountryName": "United States",
    "geoUserStateCode": "NY",
    "geoUserCity": "New York",
    "geoUserPostalCode": "10001",
    "geoUserLatitude": 40.7128,
    "geoUserLongitude": -74.0060,
    "geoUserIsAPAC": false,
    "geoUserIsEEA": false,
    "geoUserIsEU": false,
    "geoUserIsGDPR": false
  }
}
```

**Status Codes:**
- `200`: Success
- `403`: Invalid referer (security check failed)
- `500`: Internal error

---

### 2. Trial Submit Endpoint

**Endpoint:** `POST /trial-submit`

**Purpose:** Handle multi-step form submission process

#### Step 1: Initial Submission

**Request Body:**
```json
{
  "step": "1",
  "formType": "remote-support",
  "product": "Remote Support",
  "firstName": "John",
  "lastName": "Doe",
  "emailAddress": "john.doe@example.com",
  "jobRole": "IT Administrator",
  "companyName": "Acme Corp",
  "industry": "Technology",
  "turnstileToken": "0.pon7...wEgNMq",
  "sfCampaignID": "7012S0000004QbuQAE",
  "geoUserCountryCode": "US",
  "geoUserCountryIsoCode": "US",
  "geoUserCountryName": "United States",
  "geoUserStateCode": "NY",
  "geoUserPostalCode": "10001",
  "emailOptIn": true,
  "landingPageUrl": "https://www.beyondtrust.com/trial"
}
```

**Response:**
```json
{
  "success": true,
  "message": "User data received successfully.",
  "submissionId": "550e8400-e29b-41d4-a716-446655440000",
  "countryType": "allow",
  "countryBlocked": false,
  "builderStatus": "available"
}
```

**Validation Rules:**
- All required fields must be present
- Email must be valid format
- Country must be in allowlist (not blocked)
- Turnstile token must be valid
- Character validation for XSS prevention

**Builder Status Values:**
- `available`: Trial can proceed
- `constrained`: User is rate-limited
- `pending`: Manual review required (free email/no company)
- `unavailable`: Builder API unavailable

---

#### Step 2: Phone Verification

**Request Body (Send SMS/Call):**
```json
{
  "step": "2",
  "submissionId": "550e8400-e29b-41d4-a716-446655440000",
  "emailAddress": "john.doe@example.com",
  "phoneNumber": "+12125551234",
  "verificationMethod": "sms",
  "country": "United States",
  "formType": "remote-support",
  "product": "Remote Support",
  "firstName": "John",
  "lastName": "Doe",
  "jobRole": "IT Administrator",
  "companyName": "Acme Corp",
  "landingPageUrl": "https://www.beyondtrust.com/trial"
}
```

**Response (SMS Sent):**
```json
{
  "success": true,
  "message": "Verification code sent successfully.",
  "submissionId": "550e8400-e29b-41d4-a716-446655440000",
  "referenceId": "01234567890ABCDEF",
  "isFreeEmail": false,
  "builderStatus": "available"
}
```

**Request Body (Verify Code):**
```json
{
  "step": "2",
  "submissionId": "550e8400-e29b-41d4-a716-446655440000",
  "emailAddress": "john.doe@example.com",
  "phoneNumber": "+12125551234",
  "verificationCode": "123456",
  "verificationMethod": "sms",
  "country": "United States",
  "formType": "remote-support",
  "product": "Remote Support",
  "firstName": "John",
  "lastName": "Doe",
  "jobRole": "IT Administrator",
  "companyName": "Acme Corp",
  "landingPageUrl": "https://www.beyondtrust.com/trial"
}
```

**Response (Code Verified):**
```json
{
  "success": true,
  "message": "Verification successful.",
  "builderStatus": "available"
}
```

**Validation Logic:**
- Phone number format validation (max 15 chars)
- No letters or special characters except +
- Rate limiting check (email and phone)
- Free email domain detection
- Company enrichment lookup
- Submission history validation

---

#### Step 3: Final Submission

**Request Body:**
```json
{
  "step": "3",
  "submissionId": "550e8400-e29b-41d4-a716-446655440000",
  "emailAddress": "john.doe@example.com",
  "phoneNumber": "+12125551234",
  "datacenter": "US1",
  "formType": "remote-support",
  "product": "Remote Support",
  "firstName": "John",
  "lastName": "Doe",
  "jobRole": "IT Administrator",
  "companyName": "Acme Corp",
  "country": "United States",
  "verificationMethod": "sms",
  "landingPageUrl": "https://www.beyondtrust.com/trial"
}
```

**Response:**
```json
{
  "success": true,
  "message": "User data queued successfully.",
  "submissionId": "550e8400-e29b-41d4-a716-446655440000",
  "isFreeEmail": false,
  "builderStatus": "available"
}
```

**Processing:**
- Validates previous step completion
- Checks pending status from previous steps
- Generates random Salesforce Account ID
- Queues message to SQS for async processing
- Updates submission history with completion timestamp

---

### 3. Process Endpoint (Internal)

**Endpoint:** `POST /process`

**Purpose:** Process queued submissions (triggered by SQS or manually for testing)

**Trigger:**
- SQS Queue messages (production)
- API Gateway POST (local testing)

**Processing Steps:**
1. SQS message deduplication (first-writer-wins)
2. Submission record retrieval
3. Submission filtering based on flags
4. Data enrichment (6Sense, Clearbit)
5. Eloqua form submission
6. Builder API provisioning
7. Mark message as processed

---

## Data Flow

### User Journey Flow

```
┌──────────────┐
│   Step 1:    │
│   Initial    │
│  Submission  │
└──────┬───────┘
       │
       ├─► Turnstile CAPTCHA Validation
       ├─► Country Allowlist Check
       ├─► Email Rate Limit Check
       ├─► Generate submissionId + clientId
       ├─► Store in SubmissionsHistory
       ├─► Send to SQS
       │
       ▼
┌──────────────┐
│   Step 2:    │
│    Phone     │
│ Verification │
└──────┬───────┘
       │
       ├─► Validate Submission Exists
       ├─► Email Domain Check (free email detection)
       ├─► Company Enrichment Lookup
       ├─► Phone Rate Limit Check
       │
       ├──► Request Code Path:
       │    ├─► Generate 6-digit code
       │    ├─► Send via TeleSign (SMS/Call)
       │    ├─► Store code in SubmissionsHistory
       │    └─► Send to SQS
       │
       └──► Verify Code Path:
            ├─► Validate code matches stored code
            ├─► Update verification timestamp
            ├─► Send to SQS
            │
            ▼
┌──────────────┐
│   Step 3:    │
│    Final     │
│  Submission  │
└──────┬───────┘
       │
       ├─► Validate Previous Steps
       ├─► Check Pending Status
       ├─► Generate Salesforce ID
       ├─► Update completion timestamp
       ├─► Send to SQS
       │
       ▼
┌──────────────┐
│     SQS      │
│   Processor  │
└──────┬───────┘
       │
       ├─► Dedup SQS Message
       ├─► Filter by Flag (green/yellow/red)
       ├─► Enrich Data (6Sense, Clearbit)
       ├─► Submit to Eloqua
       ├─► Provision Trial (Builder API)
       └─► Mark as Processed
```

### Submission State Transitions

```
┌─────────────┐
│   Created   │ ◄─── Step 1 completed
└──────┬──────┘
       │
       ├─► countryBlocked = true ──► Blocked
       ├─► rate_limited_email ──────► Constrained
       │
       ▼
┌─────────────┐
│ Code Sent   │ ◄─── Step 2 code requested
└──────┬──────┘
       │
       ├─► free_email_detected ──────► Pending
       ├─► company_lookup_failed ────► Pending
       ├─► rate_limited_phone ───────► Constrained
       ├─► rate_limited_email_phone ─► Constrained
       │
       ▼
┌─────────────┐
│  Verified   │ ◄─── Step 2 code verified
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Completed  │ ◄─── Step 3 completed
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Processing │ ◄─── SQS picked up
└──────┬──────┘
       │
       ├─► GREEN ──► Eloqua + Builder API
       ├─► YELLOW ─► Eloqua only (pending review)
       └─► RED ────► Skip Builder API
```

---

## Database Schema

### DynamoDB Tables

#### 1. SubmissionsHistory

**Purpose:** Track all form submissions with complete audit trail

**Keys:**
- Partition Key: `clientId` (SHA-256 hash of email)
- Sort Key: `submissionId` (UUID)

**GSI:**
- PhoneNumberHashIndex: `phoneNumberHash` (HASH)

**Attributes:**
```typescript
{
  clientId: string;                      // SHA-256(email)
  submissionId: string;                  // UUID
  formName: string;                      // "rsTrial" | "praTrial"
  createdAt: number;                     // Unix timestamp
  updatedAt: number;                     // Unix timestamp
  verifiedAt?: number;                   // Unix timestamp
  completedAt?: number;                  // Unix timestamp
  turnstileValidated: boolean;           // CAPTCHA verified
  countryType: string;                   // "allow" | "blocked" | "unknown"
  countryBlocked: boolean;               // true if blocked
  phoneNumberHash?: string;              // SHA-256(phone)
  isFreeEmail?: boolean;                 // Free email domain detected
  builderStatus?: string;                // "available" | "constrained" | "pending" | "unavailable"
  submissionStatus?: string;             // Specific status code
  submissionStatusDescription?: string;  // Human-readable status
  submissionFlag?: string;               // "green" | "yellow" | "red"

  // TeleSign verification data
  telesignValidation?: {
    smsSentAt: number;
    referenceId: string;
    verificationCode: string;           // 6-digit code
  };

  // Audit trail
  stepHistory: Array<{
    step: string;                       // "1" | "2" | "3"
    timestamp: number;
    action?: string;
    apiRequestId: string;
    salesforceId?: string;              // Step 3 only
    userMetadata: {
      ipAddress: string;
      userAgent: string;
      emailOptIn?: boolean;
      geoLocation?: {
        countryCode: string;
        countryName: string;
        stateCode: string;
        postalCode: string;
        countryType: string;
        countryBlocked: boolean;
      };
    };
  }>;
}
```

**Indexes Used For:**
- Query by email: `clientId` lookup
- Query by phone: `PhoneNumberHashIndex` lookup
- Rate limiting checks
- Submission history tracking

---

#### 2. Countries

**Purpose:** Country allowlist/blocklist for geolocation-based access control

**Keys:**
- Partition Key: `country` (2-letter ISO code)
- Sort Key: `type`

**Attributes:**
```typescript
{
  country: string;  // "US", "CA", "RU", etc.
  type: string;     // "allow" | "blocked"
}
```

**Usage:**
- Step 1: Check if user's country is allowed
- Blocked countries → redirect to contact sales

**Seed Data:**
- Allowed: US, CA, BR, UK, GB, SG, DE, FR, AU, IN, RS, IL, KR, KO
- Blocked: RU, CN

---

#### 3. Domains

**Purpose:** Email domain categorization (free email, blocked domains)

**Keys:**
- Partition Key: `domain`
- Sort Key: `type`

**Attributes:**
```typescript
{
  domain: string;      // "gmail.com", "yahoo.com", etc.
  type: string;        // "free" | "blocked"
}
```

**Usage:**
- Step 2: Check if email domain is free (pending review)
- Step 2: Check if email domain is blocked (reject)

---

#### 4. ContactAccessList

**Purpose:** Allow/block specific email addresses or phone numbers

**Keys:**
- Partition Key: `contactType` ("email" | "phone")
- Sort Key: `contactValue`

**GSI:**
- ListTypeIndex: `listType` (HASH)

**Attributes:**
```typescript
{
  contactType: string;   // "email" | "phone"
  contactValue: string;  // Email address or phone number
  listType: string;      // "allow" | "block"
}
```

**Usage:**
- Allow-listed emails bypass rate limits
- Blocked contacts are rejected

---

#### 5. ApiStatus

**Purpose:** Track external API availability status

**Keys:**
- Partition Key: `apiName`
- Sort Key: `status`

**Attributes:**
```typescript
{
  apiName: string;     // "builder"
  status: string;      // "available" | "unavailable"
  requestId: string;
  statusCode: number;
  timestamp: number;
}
```

**Usage:**
- Check Builder API availability before provisioning
- Fallback to "pending" if unavailable

---

#### 6. SqsDedup

**Purpose:** SQS message deduplication using first-writer-wins pattern

**Keys:**
- Partition Key: `sqsMsgId`

**TTL:** 24 hours

**Attributes:**
```typescript
{
  sqsMsgId: string;      // SQS messageId
  status: string;        // "Processing" | "Processed"
  createdAt: number;
  processedAt?: number;
  ttl: number;          // Unix timestamp for TTL
  bodyHash: string;     // SHA-256 of message body
}
```

**Usage:**
- Prevent duplicate processing across concurrent Lambda invocations
- Conditional PUT ensures only one Lambda processes each message

---

## Validation & Business Rules

### Step 1 Validation Rules

**Required Fields:**
- `step`: Must be "1"
- `formType`: "remote-support" | "privileged-remote-access"
- `product`: String, no special chars
- `firstName`: String, no special chars
- `lastName`: String, no special chars
- `emailAddress`: Valid email format, lowercase
- `jobRole`: String, max 100 chars
- `companyName`: String, max 200 chars
- `turnstileToken`: CAPTCHA token
- `landingPageUrl`: String

**Optional Fields:**
- `industry`: String
- `sfCampaignID`: String (18 chars)
- `geoUserCountryCode`: 2-letter code
- `geoUserCountryIsoCode`: 2-letter code
- `geoUserCountryName`: String
- `geoUserStateCode`: Max 5 chars
- `geoUserPostalCode`: Max 20 chars
- `geoUserIsAPAC`: Boolean
- `geoUserIsEEA`: Boolean
- `geoUserIsEU`: Boolean
- `geoUserIsGDPR`: Boolean
- `emailOptIn`: Boolean

**Character Validation:**
- Regex 01 (allows slash): `/[<>{}[\]()\\;:|=+*^%$#@!~\`'"&]/`
- Regex 02 (no slash): `/[<>{}[\]()/\\;:|=+*^%$#@!~\`'"&]/`
- Used to prevent XSS attacks

**Business Rules:**
1. **Turnstile Validation**: Must pass Cloudflare CAPTCHA
2. **Country Check**: Country must be in allowlist
3. **Email Rate Limit**: Check if email has active submission
4. **Eloqua Form Mapping**:
   - Default: "SRAEvalSignup"
   - If country blocked: "SRAContactSales"

---

### Step 2 Validation Rules

**Additional Required Fields:**
- `submissionId`: UUID from Step 1
- `phoneNumber`: String, max 15 chars, no letters
- `verificationMethod`: "sms" | "call"
- `country`: String (optional in schema)

**Optional Fields:**
- `verificationCode`: 6-digit code (when verifying)
- `referenceId`: TeleSign reference ID

**Phone Number Validation:**
- Regex: `/[<>{}[\]()/\\;:|=*^%$#@!~\`'"&a-zA-Z]/`
- Max 15 characters (E.164 format)
- Can contain: digits, +, spaces, dashes

**Business Rules:**
1. **Submission Validation**: Must have valid Step 1 submission
2. **Turnstile Check**: Step 1 must have validated CAPTCHA
3. **Domain Check**: Detect free email domains
4. **Company Lookup**: Check 6Sense/Clearbit for company
5. **Phone Rate Limit**: Check phone number usage
6. **Combined Rate Limit**: Check email+phone combination
7. **Verification Code**: Store in database for validation

**Free Email Handling:**
- Detected → `builderStatus = "pending"`
- Requires manual review
- Status: "free_email_detected"

**Company Lookup Logic:**
1. Query 6Sense Company API
2. If no result, query Clearbit
3. If no company found → `builderStatus = "pending"`
4. Status: "company_lookup_failed"

---

### Step 3 Validation Rules

**Additional Required Fields:**
- `datacenter`: String, min 3 chars

**Auto-Generated:**
- `salesforceId`: Random 18-char ID (not real Salesforce ID)

**Business Rules:**
1. **Previous Steps**: Must have completed Steps 1 & 2
2. **Pending Check**: Cannot proceed if `builderStatus = "pending"`
3. **Builder Status Inheritance**: Use status from Step 2
4. **Eloqua Form**: "SRAEvalActivate"

---

### Rate Limiting Logic

#### Email Rate Limiting

**Rules:**
- Check submissions in SubmissionsHistory by `clientId`
- Filter: Same `formName` + `completedAt` within last 365 days
- Exclude: Current `submissionId`
- Allow-listed emails bypass rate limit

**Logic:**
```typescript
const submissions = FilterEngine.filter(allSubmissions, {
  operator: 'and',
  conditions: [
    { field: 'formName', operator: 'eq', value: formName },
    { field: 'completedAt', operator: 'gte', value: now - 365days },
    { field: 'submissionId', operator: 'ne', value: excludeId }
  ]
});

if (submissions.length > 0) {
  return { canSubmit: false, reason: 'rate_limited_email' };
}
```

---

#### Phone Rate Limiting

**Rules:**
- Query PhoneNumberHashIndex by `phoneNumberHash`
- Filter: Same `formName` + `completedAt` within last 365 days
- Exclude: Current `submissionId`

**Logic:**
```typescript
const phoneHash = createHash('sha256').update(phoneNumber).digest('hex');
const submissions = await dynamo.queryByGSI('PhoneNumberHashIndex', phoneHash);

const filtered = FilterEngine.filter(submissions, {
  operator: 'and',
  conditions: [
    { field: 'formName', operator: 'eq', value: formName },
    { field: 'completedAt', operator: 'gte', value: now - 365days },
    { field: 'submissionId', operator: 'ne', value: excludeId }
  ]
});

if (filtered.length > 0) {
  return { canSubmit: false, reason: 'rate_limited_phone' };
}
```

---

#### Combined Email+Phone Rate Limiting

**Rules:**
- Must pass both email and phone checks
- If either fails → `builderStatus = "constrained"`

---

### Submission Status Values

**Builder Status (Overall State):**
- `available`: Can provision trial immediately
- `constrained`: Rate-limited, cannot provision
- `pending`: Needs manual review
- `unavailable`: Builder API down

**Submission Status (Specific Reason):**
- `rate_limited_email`: Email has recent submission
- `rate_limited_phone`: Phone has recent submission
- `rate_limited_email_phone`: Both have recent submissions
- `free_email_detected`: Free email domain
- `company_lookup_failed`: No company found in enrichment

**Submission Flag (Processing Directive):**
- `green`: Process normally (Eloqua + Builder)
- `yellow`: Send to Eloqua, skip Builder (pending review)
- `red`: Skip Builder API (constrained/blocked)

---

## External Integrations

### 1. Cloudflare Turnstile (CAPTCHA)

**Purpose:** Bot detection and spam prevention

**Implementation:** [src/libs/turnstileClient.ts](src/libs/turnstileClient.ts)

**Configuration:**
```typescript
TURNSTILE_SECRET_KEY: string          // Secret from Cloudflare
TURNSTILE_SECRET_NAME: string         // AWS Secrets path
TURNSTILE_BYPASS_ENABLED: boolean     // Dev mode bypass
TURNSTILE_BYPASS_ALLOWED_IPS: string  // CSV of IPs
TURNSTILE_BYPASS_ALLOWED_USER_AGENTS: string
TURNSTILE_BYPASS_ALLOWED_EMAILS: string
```

**Validation Flow:**
1. Client obtains Turnstile token from frontend widget
2. Backend validates token with Cloudflare API
3. Requires matching: IP + User-Agent + Email (all three for bypass)
4. Token stored in submission history

**API Endpoint:**
```
POST https://challenges.cloudflare.com/turnstile/v0/siteverify
Body: {
  secret: string,
  response: string,  // Token from client
  remoteip: string,
  idempotency_key: string
}
```

---

### 2. TeleSign (Phone Verification)

**Purpose:** SMS/Voice verification codes

**Implementation:** [src/libs/telesignClient.ts](src/libs/telesignClient.ts)

**Configuration:**
```typescript
TELESIGN_CUSTOMER_ID: string
TELESIGN_API_KEY: string
```

**Authentication:**
```typescript
const token = Buffer.from(`${customerId}:${apiKey}`).toString('base64');
const authHeader = `Basic ${token}`;
```

**SMS Verification:**
```
POST https://rest-ww.telesign.com/v1/verify/sms
Body: {
  phone_number: string,
  verify_code: string,  // Custom 6-digit code
  is_primary: "true",
  template: "Your BeyondTrust verification code is $$CODE$$"
}
```

**Voice Call Verification:**
```
POST https://rest-ww.telesign.com/v1/verify/call
Body: {
  phone_number: string,
  tts_message: string  // "Hello, this is BeyondTrust. Your verification code is..."
}
```

**Code Generation:**
```typescript
crypto.randomInt(100000, 999999).toString();  // 6-digit code
```

**Storage:**
- Code stored in `SubmissionsHistory.telesignValidation.verificationCode`
- Reference ID stored for optional status checking

---

### 3. 6Sense Enrichment

**Purpose:** Company and person data enrichment

**Implementation:** [src/libs/enrichment.ts](src/libs/enrichment.ts)

**APIs Used:**
1. **6Sense Company API**
   - Endpoint: (from secrets)
   - Method: POST
   - Body: `{ email: string }`
   - Returns: Company data

2. **6Sense People API**
   - Endpoint: (from secrets)
   - Method: POST
   - Body: `{ email: string }`
   - Returns: Person data

3. **6Sense Identification API**
   - Endpoint: (from secrets)
   - Method: GET
   - Header: `X-6s-CustomID: {ip}`
   - Returns: Company identification from IP

**Configuration:**
```typescript
SIXSENSE_COMPANY_ENDPOINT: string
SIXSENSE_COMPANY_API_KEY: string
SIXSENSE_PEOPLE_ENDPOINT: string
SIXSENSE_PEOPLE_API_KEY: string
SIXSENSE_IDENTIFICATION_ENDPOINT: string
SIXSENSE_IDENTIFICATION_API_KEY: string
```

**Usage:**
- Step 2: Company lookup for validation
- Processor: Full enrichment for Eloqua submission

---

### 4. Clearbit Enrichment

**Purpose:** Backup company/person enrichment

**Implementation:** [src/libs/enrichment.ts](src/libs/enrichment.ts)

**API:**
```
GET {CLEARBIT_COMBINED_ENDPOINT}?email={email}
Authorization: Bearer {CLEARBIT_COMBINED_API_KEY}
```

**Configuration:**
```typescript
CLEARBIT_COMBINED_ENDPOINT: string
CLEARBIT_COMBINED_API_KEY: string
```

**Usage:**
- Fallback if 6Sense doesn't return data
- Combined person+company endpoint

---

### 5. Eloqua (Oracle Marketing Cloud)

**Purpose:** Marketing automation and lead management

**Implementation:** [src/libs/eloquaIntegration.ts](src/libs/eloquaIntegration.ts)

**Configuration:**
```typescript
ELOQUA_BASE_URL: string
ELOQUA_SITE_ID: string
```

**Form Names:**
- `SRAEvalSignup`: Step 1 (allowed countries)
- `SRAContactSales`: Step 1 (blocked countries)
- `SRAEvalValidate`: Step 2
- `SRAEvalActivate`: Step 3

**Data Enrichment Flow:**
1. Fetch enrichment data from 6Sense + Clearbit
2. Merge with form submission data
3. Calculate spam score
4. Calculate offer score (hot/warm/cold)
5. Map to Eloqua field format
6. Submit to Eloqua form endpoint

**Offer Scoring:**
- **Hot:** ContactSalesForm
- **Warm:** DemoRequest, EvaluateForm, SRA forms
- **Cold:** WhitepaperForm, OnDemandWebinarForm

**Spam Scoring:** ([src/libs/spamScoring.ts](src/libs/spamScoring.ts))
- Email validation
- Message content analysis
- Domain reputation checks
- Returns numeric score

---

### 6. Builder API (Trial Provisioning)

**Purpose:** Provision trial environments

**Implementation:** [src/libs/builderClient.ts](src/libs/builderClient.ts)

**Configuration:**
```typescript
BUILDER_API_ENDPOINT: string
BUILDER_API_KEY: string
```

**Authentication:**
```typescript
headers: {
  'Content-Type': 'application/json',
  'x-api-key': apiKey
}
```

**Submission Logic:**
1. Check ApiStatus table for Builder availability
2. If `status = "available"` → proceed
3. If `status = "unavailable"` → mark as pending
4. Submit enriched form data to Builder API
5. Return provisioning result

**Error Handling:**
- Timeout: 30 seconds
- Fallback: Mark as pending if API fails

---

### 7. MaxMind GeoIP2 (Geolocation)

**Purpose:** IP-based geolocation for country detection

**Implementation:** [src/libs/maxmindClient.ts](src/libs/maxmindClient.ts)

**Configuration:**
```typescript
MAXMIND_DB_BUCKET: string       // S3 bucket name
MAXMIND_DB_FILENAME: string     // "GeoIP2-City.mmdb"
```

**Bootstrap Flow:**
1. Load MMDB file from S3 (cached in Lambda memory)
2. Initialize MaxMind Reader
3. Extract user IP from request headers
4. Lookup IP in database
5. Return geolocation data

**Data Returned:**
```typescript
{
  geoUserCountryIsoCode: string,
  geoUserCountryName: string,
  geoUserStateCode: string,
  geoUserCity: string,
  geoUserPostalCode: string,
  geoUserLatitude: number,
  geoUserLongitude: number,
  geoUserIsAPAC: boolean,
  geoUserIsEEA: boolean,
  geoUserIsEU: boolean,
  geoUserIsGDPR: boolean
}
```

**Country Zone Detection:** ([src/libs/constants.ts](src/libs/constants.ts))
- GEO_EU: European Union countries
- GEO_EEA: European Economic Area countries
- GEO_APAC: Asia-Pacific countries
- GEO_GDPR: GDPR-applicable countries

---

## Security & Fraud Prevention

### 1. Referer Validation

**Implementation:** [src/libs/userHelper.ts](src/libs/userHelper.ts)

**Logic:**
```typescript
const ALLOWED_REFERER = [
  'beyondtrust.com',
  'localhost',
  // Additional allowed domains
];

function isAllowedReferer(referer: string): boolean {
  if (!referer) return false;
  return ALLOWED_REFERER.some(domain =>
    referer.toLowerCase().includes(domain)
  );
}
```

**Enforcement:**
- All API endpoints check referer
- Returns 403 Forbidden if invalid
- Prevents CSRF attacks

---

### 2. CAPTCHA Validation

**Implementation:** Cloudflare Turnstile

**Features:**
- Invisible CAPTCHA (low friction)
- Token expires after use
- IP + User-Agent validation
- Idempotency key for replay protection

**Bypass Mode (Development Only):**
- Requires ALL three conditions:
  - IP in allowlist
  - User-Agent in allowlist
  - Email in allowlist

---

### 3. Rate Limiting

**Multi-Dimensional Rate Limiting:**
1. **Email-based**: Max 1 submission per 365 days
2. **Phone-based**: Max 1 submission per 365 days
3. **Combined**: Both email AND phone must be unique

**Allow-list Bypass:**
- ContactAccessList with `listType = "allow"`
- Bypasses all rate limits

**Implementation:**
- DynamoDB queries with time-based filtering
- FilterEngine for complex conditions
- Phone numbers hashed with SHA-256

---

### 4. Country Blocking

**Implementation:**
- Countries table with "allow" or "blocked" type
- Bootstrap endpoint provides country data
- Step 1 validates against Countries table

**Blocked Country Handling:**
- Redirect to contact sales form
- Different Eloqua form: "SRAContactSales"
- No trial provisioning

---

### 5. Email Domain Filtering

**Free Email Detection:**
- Domains table with `type = "free"`
- Examples: gmail.com, yahoo.com, outlook.com
- Action: Mark as `pending`, require manual review

**Blocked Domains:**
- Domains table with `type = "blocked"`
- Action: Reject submission with error

**Business Email Validation:**
- If not free or blocked → assumed corporate
- Company enrichment lookup for validation

---

### 6. Spam Scoring

**Implementation:** [src/libs/spamScoring.ts](src/libs/spamScoring.ts)

**Factors:**
- Email address format and reputation
- Message content analysis
- Domain age and reputation
- Character patterns
- Suspicious keywords

**Output:**
- Numeric spam score (0-100)
- Higher score = more likely spam
- Included in Eloqua submission

---

### 7. SQS Deduplication

**Purpose:** Prevent duplicate processing

**Implementation:**
1. First-writer-wins with DynamoDB conditional PUT
2. Hash message body for content-based deduplication
3. TTL expires records after 24 hours

**Logic:**
```typescript
const bodyHash = createHash('sha256')
  .update(JSON.stringify(payload))
  .digest('hex');

const { created } = await dynamo.createItemIfNotExists(
  'SqsDedup',
  {
    sqsMsgId,
    status: 'Processing',
    ttl: now + 86400,
    bodyHash
  },
  'sqsMsgId'
);

if (!created) {
  // Another Lambda already processing
  return { proceed: false };
}
```

---

### 8. Submission Validation

**Step Sequence Enforcement:**
- Step 2 requires Step 1 completion
- Step 3 requires Step 2 completion
- Turnstile validation checked at each step

**clientId Validation:**
```typescript
const clientId = createHash('sha256')
  .update(email.toLowerCase())
  .digest('hex');

const submission = await dynamo.getItem(
  'SubmissionsHistory',
  { clientId, submissionId }
);

if (!submission || !submission.turnstileValidated) {
  return { isValid: false };
}
```

---

### 9. Data Sanitization

**Character Filtering:**
- Two regex patterns for different fields
- Prevents XSS injection
- SQL injection protection (NoSQL database)

**Email Normalization:**
- Lowercase conversion
- Trim whitespace
- Email format validation (Zod)

**Phone Number Hashing:**
- SHA-256 hash for storage
- Original number not stored in database
- GSI on phoneNumberHash for lookups

---

### 10. Audit Trail

**Comprehensive Logging:**
- All steps logged in `stepHistory` array
- IP address, User-Agent, timestamp
- Request IDs for correlation
- Geolocation data

**Winston Logger:**
- Structured JSON logging
- CloudWatch Logs integration
- Error stack traces
- Sensitive data masking

**Example Audit Entry:**
```typescript
{
  step: "2",
  timestamp: 1709664000,
  action: "sms_sent",
  apiRequestId: "abc-123",
  userMetadata: {
    ipAddress: "192.0.2.1",
    userAgent: "Mozilla/5.0...",
    geoLocation: {
      countryCode: "US",
      countryName: "United States",
      stateCode: "NY",
      postalCode: "10001"
    }
  }
}
```

---

## Form Processing Logic

### Submission Filtering

**Implementation:** [src/libs/submissionFilters.ts](src/libs/submissionFilters.ts)

**Filter Engine:**
```typescript
class FilterEngine {
  static filter<T>(items: T[], filter: Filter<T>): T[];
}

interface Filter {
  operator: 'and' | 'or';
  conditions: Array<{
    field: keyof T;
    operator: 'eq' | 'ne' | 'gt' | 'gte' | 'lt' | 'lte' | 'contains' | 'in' | 'notIn';
    value: any;
  }>;
}
```

**Usage Example:**
```typescript
const activeSubmissions = FilterEngine.filter(submissions, {
  operator: 'and',
  conditions: [
    { field: 'formName', operator: 'eq', value: 'rsTrial' },
    { field: 'completedAt', operator: 'gte', value: now - 365days },
    { field: 'submissionId', operator: 'ne', value: currentId }
  ]
});
```

**Predefined Filters:**
```typescript
class SubmissionFilters {
  static matchingSubmissionsByTimeRange(
    formName: string,
    phoneHash?: string,
    excludeId?: string
  ): Filter;

  static recentSubmissions(days: number): Filter;

  static byBuilderStatus(status: string): Filter;

  static bySubmissionFlag(flag: string): Filter;
}
```

---

### Salesforce Campaign ID Logic

**Implementation:** [src/functions/trial-handler/utils.ts](src/functions/trial-handler/utils.ts)

**Logic:**
```typescript
function setSalesforceCampaignId(submission: Record<string, any>): string {
  // 1. Use campid from query if valid (18 chars)
  if (submission.campid && /^[a-zA-Z0-9]{18}$/.test(submission.campid)) {
    return submission.campid;
  }

  // 2. Extended trial campaigns
  if (submission.trialExpirationType === 'extended' ||
      submission.trialExpirationType === 'month') {
    if (submission.formType === 'remote-support') {
      return '7017V000001NCLUQA4';
    }
    if (submission.formType === 'privileged-remote-access') {
      return '7017V000001NCLPQA4';
    }
  }

  // 3. Standard trial campaigns
  if (submission.formType === 'remote-support') {
    return '7012S0000004QbpQAE';
  }
  if (submission.formType === 'privileged-remote-access') {
    return '7012S0000004QbuQAE';
  }

  return '';
}
```

**Campaign Mapping:**
- RS Trial (Standard): `7012S0000004QbpQAE`
- PRA Trial (Standard): `7012S0000004QbuQAE`
- RS Trial (Extended): `7017V000001NCLUQA4`
- PRA Trial (Extended): `7017V000001NCLPQA4`

---

### Random Salesforce ID Generation

**Purpose:** Generate fake Salesforce Account ID for tracking

**Implementation:**
```typescript
function createRandomSalesforceId(): string {
  const letters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';

  // Start with letter
  const firstChar = letters[Math.floor(Math.random() * letters.length)];

  // Add timestamp (base 36, 7 chars)
  const timestamp = Date.now().toString(36)
    .padStart(7, '0')
    .slice(0, 7)
    .toUpperCase();

  // Add 10 random chars
  let randomPart = '';
  for (let i = 0; i < 10; i++) {
    randomPart += chars[Math.floor(Math.random() * chars.length)];
  }

  // Total: 1 + 7 + 10 = 18 chars
  return firstChar + timestamp + randomPart;
}
```

**Example Output:** `A12ABC7XDEFGHIJKLM`

---

### Eloqua Field Mapping

**Implementation:** [src/libs/eloquaIntegration.ts](src/libs/eloquaIntegration.ts)

**Enrichment Priority:**
1. Original form data (highest priority)
2. 6Sense data
3. Clearbit data
4. Calculated fields (spam score, offer score)

**Mapping Process:**
1. Collect enrichment data (6Sense + Clearbit)
2. Assign all payload data to base object
3. Override Eloqua-specific fields
4. Add enrichment data with priority logic
5. Calculate scores
6. Map geographic data
7. Add activity comments

**Key Fields:**
```typescript
{
  // Eloqua System Fields
  elqSiteID: string,
  elqFormName: string,
  elqCookieWrite: string,
  elqCustomerGUID: string,

  // User Data
  emailAddress: string,
  firstName: string,
  lastName: string,
  companyName: string,
  jobRole: string,
  phoneNumber: string,

  // Enrichment Data
  company: string,              // From 6Sense/Clearbit
  companyWebsite: string,
  companyRevenue: string,
  companyEmployees: string,
  companyIndustry: string,
  companyAddress: string,

  // Scoring
  spamScore: number,
  offerScore: string,           // "hot" | "warm" | "cold"

  // Geographic
  country: string,
  stateProvince: string,
  city: string,
  postalCode: string,

  // Tracking
  sfCampaignID: string,
  salesforceId: string,
  submissionId: string,
  leadSource: string,

  // Metadata
  activityComments: string,     // Detailed submission info
  timestamp: number
}
```

---

## Environment Configuration

### AWS Secrets Manager Structure

**Secrets by Environment:**
```
/{environment}/eloqua
/{environment}/sixsense
/{environment}/clearbit
/{environment}/telesign
/{environment}/builder
/{environment}/turnstile  (optional, can use env var)
```

**Example: `/dev/eloqua`**
```json
{
  "ELOQUA_BASE_URL": "https://s1234567890.t.eloqua.com/e/f2",
  "ELOQUA_SITE_ID": "1234567890"
}
```

**Example: `/dev/sixsense`**
```json
{
  "SIXSENSE_COMPANY_ENDPOINT": "https://api.6sense.com/v1/company",
  "SIXSENSE_COMPANY_API_KEY": "sk_...",
  "SIXSENSE_PEOPLE_ENDPOINT": "https://api.6sense.com/v1/people",
  "SIXSENSE_PEOPLE_API_KEY": "sk_...",
  "SIXSENSE_IDENTIFICATION_ENDPOINT": "https://api.6sense.com/v1/identify",
  "SIXSENSE_IDENTIFICATION_API_KEY": "sk_..."
}
```

**Example: `/dev/clearbit`**
```json
{
  "CLEARBIT_COMBINED_ENDPOINT": "https://person-stream.clearbit.com/v2/combined/find",
  "CLEARBIT_COMBINED_API_KEY": "sk_..."
}
```

**Example: `/dev/telesign`**
```json
{
  "TELESIGN_CUSTOMER_ID": "12345678-1234-1234-1234-123456789012",
  "TELESIGN_API_KEY": "abcdef..."
}
```

**Example: `/dev/builder`**
```json
{
  "BUILDER_API_ENDPOINT": "https://builder-api.beyondtrust.com",
  "BUILDER_API_KEY": "bt_..."
}
```

---

### Environment Variables

**Global (template.yaml):**
```yaml
Environment:
  Variables:
    AWS_REGION: us-east-2
    ENVIRONMENT: dev  # or staging, prod
    DYNAMODB_ENDPOINT: http://host.docker.internal:8000  # local only
    DYNAMODB_REGION: us-east-2
    DYNAMODB_ACCESS_KEY_ID: local  # local only
    DYNAMODB_SECRET_ACCESS_KEY: local  # local only
```

**Per-Function:**
```yaml
BootstrapFunction:
  Environment:
    Variables:
      DOMAINS_TABLE: Domains
      APISTATUS_TABLE: ApiStatus
      SUBMISSIONS_TABLE: SubmissionsHistory
      SECRET_ARN: !Ref MyAppSecret
      MAXMIND_DB_BUCKET: maxmind-db
      MAXMIND_DB_FILENAME: GeoIP2-City.mmdb

TrialSubmitFunction:
  Environment:
    Variables:
      DOMAINS_TABLE: Domains
      APISTATUS_TABLE: ApiStatus
      SUBMISSIONS_TABLE: SubmissionsHistory
      TRIAL_QUEUE_URL: !Ref TrialQueue
      SECRET_ARN: !Ref MyAppSecret
      TURNSTILE_SECRET_KEY: (from secrets or env)
      TURNSTILE_BYPASS_ENABLED: "false"
      TURNSTILE_BYPASS_ALLOWED_IPS: ""
      TURNSTILE_BYPASS_ALLOWED_USER_AGENTS: ""
      TURNSTILE_BYPASS_ALLOWED_EMAILS: ""

TrialProcessorFunction:
  Environment:
    Variables:
      SECRET_ARN: !Ref MyAppSecret
```

---

### Local Development Configuration

**File: `.env.json`** (not in repo, copy from `.env.json.example`)

```json
{
  "BootstrapFunction": {},
  "TrialSubmitFunction": {
    "SQS_TRIAL_QUEUE_URL": "http://host.docker.internal:9324/000000000000/TrialQueue",
    "TURNSTILE_BYPASS_ENABLED": "true",
    "TURNSTILE_BYPASS_ALLOWED_IPS": "127.0.0.1",
    "TURNSTILE_BYPASS_ALLOWED_USER_AGENTS": "Cypress 123",
    "TURNSTILE_BYPASS_ALLOWED_EMAILS": "test@test.com",
    "AWS_ACCESS_KEY_ID": "get-from-1password",
    "AWS_SECRET_KEY": "get-from-1password",
    "AWS_SESSION_TOKEN": "get-from-1password"
  },
  "TrialProcessorFunction": {
    "AWS_ACCESS_KEY_ID": "get-from-1password",
    "AWS_SECRET_KEY": "get-from-1password",
    "AWS_SESSION_TOKEN": "get-from-1password"
  }
}
```

---

### Docker Compose Services

**File: `docker-compose.yaml`**

**Services:**
1. **DynamoDB Local** (port 8000)
2. **DynamoDB Admin** (port 8001)
3. **ElasticMQ (SQS)** (port 9324)
4. **ElasticMQ UI** (port 3999)

**Setup:**
```bash
# Start services
docker-compose up -d

# Create tables and seed data
npm run dynamo:seed:local

# Access UIs
# DynamoDB: http://localhost:8001/
# SQS: http://localhost:3999/
```

---

## Development Setup

### Prerequisites

```bash
brew install aws-sam-cli     # AWS SAM CLI
brew install pre-commit      # Git hooks
# Docker Desktop v4.9.1 (known working version for macOS with ZScaler)
# Node.js >= 22
```

---

### Installation Steps

```bash
# 1. Clone repository
git clone <repo-url>
cd mkt-beyondtrust-forms-backend

# 2. Install dependencies
npm install

# 3. Set up pre-commit hooks
pre-commit install

# 4. Configure environment
cp .env.json.example .env.json
# Edit .env.json with your AWS credentials

# 5. Start Docker services
docker-compose up -d

# 6. Seed local database
npm run dynamo:seed:local

# 7. Create SQS queue
# Open http://localhost:3999/
# Create queue: TrialQueue

# 8. Start development
# Terminal 1: Build & watch
npm run dev

# Terminal 2: SAM local API
sam local start-api --template template.yaml --env-vars .env.json --host 0.0.0.0 --port 3000
```

---

### Available Scripts

```json
{
  "dev": "Build + lint + watch",
  "build": "Production build",
  "lint": "ESLint with auto-fix",
  "lint-only": "ESLint without fix",
  "prettier": "Format code",
  "tests": "Jest with coverage",
  "sam": "Start SAM local",
  "dynamo:seed:local": "Seed local DynamoDB"
}
```

---

### Testing Endpoints

**Bootstrap:**
```bash
curl http://localhost:3000/bootstrap
```

**Step 1:**
```bash
curl -X POST http://localhost:3000/trial-submit \
  -H "Content-Type: application/json" \
  -d '{
    "step": "1",
    "formType": "remote-support",
    "product": "Remote Support",
    "firstName": "Test",
    "lastName": "User",
    "emailAddress": "test@example.com",
    "jobRole": "Administrator",
    "companyName": "Test Corp",
    "industry": "Technology",
    "turnstileToken": "bypass-token",
    "sfCampaignID": "7012S0000004QbpQAE",
    "geoUserCountryCode": "US",
    "geoUserCountryIsoCode": "US",
    "geoUserCountryName": "United States",
    "geoUserStateCode": "NY",
    "geoUserPostalCode": "10001",
    "emailOptIn": true,
    "landingPageUrl": "https://www.beyondtrust.com/trial"
  }'
```

---

### Database Inspection

**DynamoDB Admin UI:**
- URL: http://localhost:8001/
- Tables: SubmissionsHistory, Countries, Domains, ApiStatus, ContactAccessList, SqsDedup

**Query Examples:**
```javascript
// In DynamoDB Admin console
// Query SubmissionsHistory by clientId
Table: SubmissionsHistory
Partition Key: clientId = "abc123..."

// Query by phoneNumberHash (GSI)
Index: PhoneNumberHashIndex
Partition Key: phoneNumberHash = "def456..."
```

---

### SQS Management

**SQS Admin UI:**
- URL: http://localhost:3999/
- Queue URL: http://host.docker.internal:9324/000000000000/TrialQueue

**Create Queue:**
1. Open http://localhost:3999/
2. Click "Create Queue"
3. Queue Name: `TrialQueue`
4. Use default settings

**View Messages:**
- Check queue depth
- View message bodies
- Purge queue if needed

---

### Debugging Tips

**CloudWatch Logs (Local):**
```bash
# SAM outputs logs to console
# Look for [INFO], [WARN], [ERROR] prefixes
```

**Common Issues:**

1. **DynamoDB Connection Failed**
   ```bash
   # Check Docker is running
   docker ps
   # Restart DynamoDB
   docker-compose restart dynamodb-local
   ```

2. **SQS Connection Failed**
   ```bash
   # Verify queue exists
   curl http://localhost:3999/
   # Create queue if missing
   ```

3. **Secrets Manager Error (Local)**
   ```bash
   # Set AWS credentials in .env.json
   # Or use environment variables
   export AWS_ACCESS_KEY_ID=...
   export AWS_SECRET_ACCESS_KEY=...
   export AWS_SESSION_TOKEN=...
   ```

4. **Turnstile Bypass Not Working**
   ```bash
   # Check all three conditions in .env.json:
   TURNSTILE_BYPASS_ENABLED=true
   TURNSTILE_BYPASS_ALLOWED_IPS=127.0.0.1
   TURNSTILE_BYPASS_ALLOWED_USER_AGENTS=<your-user-agent>
   TURNSTILE_BYPASS_ALLOWED_EMAILS=test@test.com
   ```

---

## Key Dependencies

### Production Dependencies

```json
{
  "@aws-sdk/client-dynamodb": "^3.620.0",
  "@aws-sdk/client-s3": "^3.820.0",
  "@aws-sdk/client-secrets-manager": "^3.817.0",
  "@aws-sdk/client-sqs": "^3.817.0",
  "@aws-sdk/lib-dynamodb": "^3.590.0",
  "@maxmind/geoip2-node": "^6.1.0",
  "axios": "^1.9.0",
  "date-fns": "^4.1.0",
  "sanitize-html": "^2.13.1",
  "winston": "^3.13.0",
  "zod": "^3.25.32"
}
```

**Key Libraries:**
- **Zod:** Schema validation (replaces Joi)
- **Winston:** Structured logging
- **Axios:** HTTP client for external APIs
- **MaxMind GeoIP2:** IP geolocation
- **AWS SDK v3:** Cloud services
- **sanitize-html:** XSS prevention

---

### Development Dependencies

```json
{
  "@types/node": "^20.17.12",
  "@typescript-eslint/eslint-plugin": "^6.18.1",
  "eslint": "^8.57.1",
  "jest": "^29.7.0",
  "prettier": "^3.1.1",
  "typescript": "^5.3.3",
  "webpack": "^5.x",
  "webpack-cli": "^5.1.4"
}
```

---

## File Structure

```
mkt-beyondtrust-forms-backend/
├── src/
│   ├── functions/
│   │   ├── bootstrap/
│   │   │   └── index.ts                    # Bootstrap Lambda handler
│   │   ├── trial-handler/
│   │   │   ├── index.ts                    # Trial submit handler
│   │   │   ├── constants.ts                # DB tables, error codes, statuses
│   │   │   └── utils.ts                    # Validation, response helpers
│   │   └── trial-processor/
│   │       ├── index.ts                    # SQS processor handler
│   │       └── utils.ts                    # Processing utilities
│   │
│   └── libs/
│       ├── builderClient.ts                # Builder API integration
│       ├── corsHelper.ts                   # CORS headers
│       ├── cryptoHelper.ts                 # PHP-compatible decryption
│       ├── dynamoClient.ts                 # DynamoDB wrapper
│       ├── eloquaIntegration.ts            # Eloqua API + enrichment
│       ├── enrichment.ts                   # 6Sense + Clearbit
│       ├── evalHelper.ts                   # Rate limiting logic
│       ├── filterEngine.ts                 # Application-level filtering
│       ├── logger.ts                       # Winston logger
│       ├── maxmindClient.ts                # GeoIP2 integration
│       ├── s3Client.ts                     # S3 operations
│       ├── secretsClient.ts                # Secrets Manager
│       ├── spamScoring.ts                  # Anti-spam logic
│       ├── sqsClient.ts                    # SQS operations
│       ├── submissionFilters.ts            # Predefined filters
│       ├── telesignClient.ts               # Phone verification
│       ├── turnstileClient.ts              # CAPTCHA validation
│       ├── userHelper.ts                   # User utilities
│       └── constants.ts                    # Global constants
│
├── scripts/
│   └── seed-dynamodb-local.js              # Local DB seeder
│
├── dist/                                   # Webpack output (Lambda packages)
├── docker/                                 # Docker configs
├── s3/                                     # Local S3 files
│
├── template.yaml                           # AWS SAM template
├── webpack.config.js                       # Webpack config
├── tsconfig.json                           # TypeScript config
├── package.json                            # Dependencies
├── .env.json.example                       # Environment template
├── docker-compose.yaml                     # Local services
└── README.md                               # Setup instructions
```

---

## Form Type Mapping

### Form Types

**Frontend `formType` Values:**
- `remote-support` → Backend: `rsTrial`
- `privileged-remote-access` → Backend: `praTrial`

**Mapping Function:**
```typescript
function getFormNameFromType(formType: string): string {
  switch (formType) {
    case 'remote-support':
      return 'rsTrial';
    case 'privileged-remote-access':
      return 'praTrial';
    default:
      return 'rsTrial';  // Fallback
  }
}
```

**Usage:**
- Rate limiting checks by `formName`
- Separate trial limits per product

---

## Error Handling

### Error Codes

```typescript
const ERROR_CODES = {
  VALIDATION_ERROR: 'VALIDATION_ERROR',
  BLOCKLISTED_DOMAIN: 'BLOCKLISTED_DOMAIN',
  INVALID_SUBMISSION: 'INVALID_SUBMISSION',
  INVALID_VERIFICATION_CODE: 'INVALID_VERIFICATION_CODE',
  INVALID_STEP: 'INVALID_STEP',
  SQS_ERROR: 'SQS_ERROR',
  INTERNAL_ERROR: 'INTERNAL_ERROR',
  TURNSTILE_VALIDATION_FAILED: 'TURNSTILE_VALIDATION_FAILED'
};
```

### Error Response Format

```json
{
  "success": false,
  "message": "Validation failed for step 1",
  "errorCode": "VALIDATION_ERROR",
  "errors": {
    "emailAddress": "Invalid email format",
    "firstName": "First name is required"
  },
  "step": "1"
}
```

### Success Response Format

```json
{
  "success": true,
  "message": "User data received successfully.",
  "submissionId": "550e8400-e29b-41d4-a716-446655440000",
  "countryType": "allow",
  "countryBlocked": false,
  "builderStatus": "available"
}
```

---

## YAML Form System Considerations

### Current Form Structure

**Step 1 Fields:**
- product, firstName, lastName, emailAddress
- jobRole, companyName, industry
- turnstileToken, sfCampaignID
- geoUser* (geolocation fields)
- emailOptIn, landingPageUrl, formType

**Step 2 Fields:**
- phoneNumber, verificationMethod, country
- verificationCode (when verifying)
- All Step 1 fields repeated

**Step 3 Fields:**
- datacenter
- All Step 1 & 2 fields repeated

### YAML Configuration Recommendations

**Form Definition Structure:**
```yaml
formType: remote-support
version: 1.0.0

steps:
  - id: 1
    name: "Initial Information"
    eloquaForm: "SRAEvalSignup"
    fields:
      - name: firstName
        type: text
        required: true
        validation:
          maxLength: 100
          pattern: "^[^<>{}\\[\\]()\\\\;:|=+*^%$#@!~`'\"&/]*$"

      - name: emailAddress
        type: email
        required: true
        validation:
          format: email

      - name: companyName
        type: text
        required: true
        validation:
          maxLength: 200

      # ... other fields

    validations:
      - type: turnstile
        required: true
      - type: country
        table: Countries
        action: checkAllowlist
      - type: rateLimit
        dimension: email
        period: 365days

  - id: 2
    name: "Phone Verification"
    eloquaForm: "SRAEvalValidate"
    inheritsFields: [1]  # Inherit all fields from Step 1
    fields:
      - name: phoneNumber
        type: tel
        required: true
        validation:
          maxLength: 15
          pattern: "^[0-9\\+\\-\\s]*$"

      - name: verificationMethod
        type: select
        required: true
        options: [sms, call]

      - name: verificationCode
        type: text
        required: false  # Only required when verifying
        validation:
          length: 6

    validations:
      - type: submissionExists
        requireTurnstile: true
      - type: emailDomain
        table: Domains
        checkFree: true
      - type: enrichment
        provider: [6sense, clearbit]
        action: companyLookup
      - type: rateLimit
        dimension: phone
        period: 365days
      - type: rateLimit
        dimension: combined
        fields: [email, phone]
        period: 365days

  - id: 3
    name: "Final Configuration"
    eloquaForm: "SRAEvalActivate"
    inheritsFields: [1, 2]  # Inherit from both steps
    fields:
      - name: datacenter
        type: select
        required: true
        validation:
          minLength: 3

    validations:
      - type: submissionComplete
        requiredSteps: [1, 2]
      - type: builderStatus
        allowedStatuses: [available, constrained]

processors:
  - type: sqs
    queue: TrialQueue
    batchSize: 5

  - type: enrichment
    providers:
      - name: 6sense
        apis: [company, people, identification]
      - name: clearbit
        apis: [combined]

  - type: scoring
    types: [spam, offer]

  - type: provisioning
    conditions:
      - field: submissionFlag
        operator: eq
        value: green
    api: builder

  - type: marketing
    api: eloqua
    mapping: eloquaFields.yaml
```

### YAML Benefits for This System

1. **Declarative Field Definitions**
   - Type-safe field declarations
   - Built-in validation rules
   - No code changes for field additions

2. **Step Configuration**
   - Field inheritance between steps
   - Eloqua form mapping per step
   - Conditional field display

3. **Validation Rules**
   - External service validations (Turnstile, TeleSign)
   - Database checks (Countries, Domains)
   - Rate limiting configuration
   - Business rule definitions

4. **Multi-Form Support**
   - Same backend, different forms
   - Per-form rate limiting
   - Campaign ID mapping

5. **Processing Pipeline**
   - Configurable enrichment providers
   - Conditional provisioning
   - Flag-based filtering

6. **Maintainability**
   - Non-developers can modify forms
   - Version control for form schemas
   - A/B testing support

### Migration Path

**Phase 1: YAML Schema Definition**
- Define YAML schema for forms
- Implement YAML parser
- Create validation engine from YAML
- Test with existing forms

**Phase 2: Backend Adaptation**
- Update validation logic to consume YAML
- Make field handling dynamic
- Externalize business rules
- Preserve backward compatibility

**Phase 3: Frontend Integration**
- Generate frontend forms from YAML
- Dynamic field rendering
- Client-side validation from schema
- Progressive enhancement

**Phase 4: Advanced Features**
- A/B testing support
- Multi-language forms
- Conditional field display
- Dynamic field dependencies

---

## Security Best Practices

1. **Never Log Sensitive Data**
   - Email addresses masked in logs
   - Phone numbers hashed
   - API keys redacted
   - Use `maskEmail()` helper

2. **Secrets Management**
   - All secrets in AWS Secrets Manager
   - No hardcoded credentials
   - Rotate keys regularly
   - Environment-specific secrets

3. **Input Validation**
   - Zod schema validation
   - Character filtering (XSS prevention)
   - Length limits enforced
   - Type checking

4. **Rate Limiting**
   - Multi-dimensional checks
   - Time-based windows
   - Allow-list bypass
   - Audit trail logging

5. **CAPTCHA Protection**
   - Turnstile on all submissions
   - Token validation before processing
   - IP + User-Agent verification
   - Idempotency keys

6. **Database Security**
   - Hashed identifiers (email, phone)
   - No PII in partition keys
   - TTL for temporary data
   - Least privilege IAM policies

---

## Performance Optimizations

1. **Lambda Cold Start Reduction**
   - Module-level caching
   - Lazy initialization
   - Connection pooling
   - S3 buffer caching (MaxMind)

2. **Database Efficiency**
   - GSI for phone lookups
   - Batch operations where possible
   - Conditional writes (dedup)
   - TTL for automatic cleanup

3. **API Call Optimization**
   - Parallel enrichment calls
   - Graceful degradation
   - Timeout configurations
   - Retry logic

4. **SQS Processing**
   - Batch size: 5 messages
   - First-writer-wins dedup
   - Parallel processing
   - Dead letter queue for failures

---

## Monitoring & Observability

### CloudWatch Metrics

**Lambda Metrics:**
- Invocations
- Errors
- Duration
- Throttles
- Concurrent executions

**DynamoDB Metrics:**
- Consumed read/write capacity
- Throttled requests
- User errors

**SQS Metrics:**
- Messages sent
- Messages received
- Messages deleted
- Queue depth
- Age of oldest message

### Custom Metrics (Recommended)

```typescript
// Step completion rates
putMetric('StepCompletionRate', {
  step: '1',
  formType: 'rsTrial',
  value: 1
});

// Rate limiting hits
putMetric('RateLimitHit', {
  dimension: 'email',
  formType: 'praTrial',
  value: 1
});

// Enrichment failures
putMetric('EnrichmentFailure', {
  provider: '6sense',
  api: 'company',
  value: 1
});

// Builder API status
putMetric('BuilderApiStatus', {
  status: 'available',
  value: 1
});
```

### Logging Best Practices

**Structured Logging:**
```typescript
logger.info('Step 1 completed', {
  requestId: 'abc-123',
  submissionId: 'def-456',
  formType: 'rsTrial',
  email: maskEmail('user@example.com'),
  country: 'US',
  builderStatus: 'available'
});
```

**Log Levels:**
- `info`: Normal operations
- `warn`: Recoverable errors (e.g., enrichment API down)
- `error`: Critical failures requiring investigation

### Alerting

**Critical Alerts:**
- Lambda error rate > 5%
- DynamoDB throttling
- SQS queue depth > 1000
- Builder API unavailable > 5 minutes
- Turnstile validation rate < 95%

**Warning Alerts:**
- Step 2 verification rate < 80%
- Step 3 completion rate < 60%
- Enrichment API failures > 10%
- Rate limiting hits increasing

---

## Appendix

### Glossary

- **clientId:** SHA-256 hash of email address (partition key)
- **submissionId:** UUID for unique submission tracking
- **formName:** Backend identifier (rsTrial, praTrial)
- **formType:** Frontend identifier (remote-support, privileged-remote-access)
- **builderStatus:** Trial provisioning state (available, constrained, pending, unavailable)
- **submissionStatus:** Specific reason for status (rate_limited_email, etc.)
- **submissionFlag:** Processing directive (green, yellow, red)
- **phoneNumberHash:** SHA-256 hash of phone number
- **turnstileToken:** Cloudflare CAPTCHA token
- **sfCampaignID:** Salesforce Campaign ID for tracking
- **salesforceId:** Fake Salesforce Account ID (18 chars)
- **elqFormName:** Eloqua form endpoint name

### Related Documentation

- [AWS SAM Developer Guide](https://docs.aws.amazon.com/serverless-application-model/)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [Zod Documentation](https://zod.dev/)
- [Winston Logger](https://github.com/winstonjs/winston)
- [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/)
- [TeleSign API](https://developer.telesign.com/)

### Contact

For questions or issues related to this documentation or the backend system:
- Open an issue in the repository
- Contact the development team
- See README.md for setup instructions

---

**Document Version:** 1.0
**Last Updated:** 2025-01-30
**Maintained By:** BeyondTrust Development Team
