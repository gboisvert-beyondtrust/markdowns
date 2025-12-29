# Low Level Design: YAML-Driven Form Builder System

**Project**: BeyondTrust Forms Frontend Evolution
**Version**: 1.0.0
**Date**: 2025-10-21
**Status**: Design Phase

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Architecture Analysis](#2-current-architecture-analysis)
3. [Target Architecture](#3-target-architecture)
4. [YAML Schema Design](#4-yaml-schema-design)
5. [Form Rendering Engine](#5-form-rendering-engine)
6. [Migration Strategy](#6-migration-strategy)
7. [Technical Specifications](#7-technical-specifications)
8. [Implementation Plan](#8-implementation-plan)
9. [Risk Assessment](#9-risk-assessment)
10. [Appendices](#10-appendices)

---

## 1. Executive Summary

### 1.1 Vision

Transform the BeyondTrust forms frontend from a route-based, hardcoded form system into a scalable, configuration-driven form builder that can support all website forms while maintaining excellent performance and developer experience.

### 1.2 Goals

- **Scalability**: Support unlimited forms without creating new routes or components for each form
- **Performance**: Generate flat HTML with minimal JavaScript for fast initial loads
- **Maintainability**: Use YAML/Markdown for form definitions, reducing code complexity
- **Flexibility**: Support various form types (trial, contact, demo, event registration, etc.)
- **Type Safety**: Maintain TypeScript type checking despite configuration-driven approach
- **Progressive Enhancement**: Start with functional HTML, enhance with Next.js features

### 1.3 Success Metrics

- Reduce time to create new form from X hours to Y minutes
- Maintain or improve Core Web Vitals scores
- Support 10+ form types within 6 months
- Zero regression in existing trial form functionality

---

## 2. Current Architecture Analysis

### 2.1 Overview

The current system is built on Next.js 15 with TypeScript, supporting two trial products (Remote Support and Privileged Remote Access) in three rendering modes:

1. **Standard Page** (`/trial/remote-support`) - Full page with header/footer
2. **Single Page** (`/trial/remote-support/single`) - Unified form for modal embedding
3. **iFrame** (`/trial/remote-support/iframe`) - Embeddable version

### 2.2 Component Hierarchy

```
src/
├── pages/trial/
│   ├── remote-support.tsx                    # Route: /trial/remote-support
│   ├── remote-support/
│   │   ├── single.tsx                        # Route: /trial/remote-support/single
│   │   └── iframe.tsx                        # Route: /trial/remote-support/iframe
│   ├── privileged-remote-access.tsx          # Route: /trial/privileged-remote-access
│   └── privileged-remote-access/
│       ├── single.tsx
│       └── iframe.tsx
├── components/forms/trial/
│   ├── TrialForm.tsx                         # Multi-step (stepped) form orchestrator
│   ├── UnifiedTrialForm.tsx                  # Single-page form orchestrator
│   ├── TrialFormContext.tsx                  # Form type & mode context provider
│   ├── steps/                                # Step components (for stepped mode)
│   │   ├── UserProfileSteps.tsx              # Step 1: User info
│   │   ├── PhoneValidationStep.tsx           # Step 2: Phone verification
│   │   ├── DatacenterSelectionStep.tsx       # Step 3: Datacenter selection
│   │   ├── SuccessStep.tsx                   # Final success screen
│   │   └── WarningStep.tsx                   # Restriction/error screen
│   └── fields/                               # Reusable field components
│       ├── UserProfileFields.tsx             # Name, email, job role, company, industry
│       ├── PhoneValidationFields.tsx         # International phone input
│       └── DatacenterSelectionFields.tsx     # Region selection dropdown
├── components/forms/
│   ├── FormField.tsx                         # Wrapper with label & error display
│   ├── InputField.tsx                        # Text input with validation styling
│   ├── SelectField.tsx                       # Dropdown with validation styling
│   ├── PhoneInputWithButtons.tsx             # Phone input with SMS/Call buttons
│   └── VerificationCodeInput.tsx             # 6-digit verification code input
├── components/ui/
│   ├── Button.tsx                            # Primary button component
│   ├── Card.tsx                              # Container card
│   ├── ProgressSteps.tsx                     # Multi-step progress indicator
│   ├── ErrorAlert.tsx                        # Error message display
│   └── ...
├── config/
│   ├── trial.ts                              # Product configurations
│   └── env.ts                                # Environment configuration
├── types/
│   ├── forms.ts                              # Form data type definitions
│   ├── trial.ts                              # Trial-specific types
│   └── api.ts                                # API response types
├── services/
│   ├── api.ts                                # API client for form submission
│   └── bootstrapService.ts                   # Geolocation data fetching
└── utils/
    ├── validation.ts                         # Client-side validation rules
    ├── geolocationUtils.ts                   # Geolocation data extraction
    ├── form-events.ts                        # Analytics event tracking
    └── ...
```

### 2.3 Key Patterns & Architecture Decisions

#### 2.3.1 Form Modes

**Stepped Mode** (`TrialForm.tsx`):
- Multi-step wizard (3 steps)
- Progress indicator at top
- Back navigation enabled
- Each step validates before proceeding
- API submission per step

**Unified Mode** (`UnifiedTrialForm.tsx`):
- All fields on single page
- No step progression
- Simpler validation flow
- Single API submission at end

#### 2.3.2 Context Architecture

```typescript
// src/components/forms/trial/TrialFormContext.tsx
export type FormMode = 'stepped' | 'unified';
export type FormType = 'remote-support' | 'privileged-remote-access';

<FormTypeProvider formType={formType} formMode={mode}>
  <TrialForm />
</FormTypeProvider>
```

#### 2.3.3 Type System

```typescript
// src/types/forms.ts
export interface UserProfileData {
  firstName: string;
  lastName: string;
  emailAddress: string;
  jobRole: string;
  companyName: string;
  industry: string;
  turnstileToken?: string;
  emailOptIn?: boolean;
  campid?: string;
  landingPageUrl?: string;
}

export interface PhoneValidationData {
  phoneNumber: string;
  verificationCode: string;
  verificationMethod: 'sms' | 'call' | null;
}

export interface DatacenterData {
  datacenter: string;
}

export interface GeoLocationData {
  geoUserCountryIsoCode?: string;
  geoUserIsGDPR?: boolean;
  // ... 20+ geolocation fields
}

export interface FormData extends
  UserProfileData,
  PhoneValidationData,
  DatacenterData,
  GeoLocationData {}
```

#### 2.3.4 Configuration Pattern

```typescript
// src/config/trial.ts
export interface ProductConfig {
  buttonLabel: string;
  productUrl: string;
  pageTitle: string;
  pageSubtitle: string;
  title?: string;
  description?: string;
}

export const PRODUCT_CONFIGS: Record<FormType, ProductConfig> = {
  'remote-support': { /* config */ },
  'privileged-remote-access': { /* config */ }
};
```

#### 2.3.5 Reusable Field Components

Field components accept:
- Current field values
- Error states
- onChange handlers
- Form mode ('stepped' | 'unified')
- Initial data (for geolocation, pre-population)

Example: `DatacenterSelectionFields.tsx`
```typescript
interface DatacenterSelectionFieldsProps {
  datacenter: string;
  error: string;
  formMode?: FormMode;
  initialData?: PartialFormData;
  onChange: (e: React.ChangeEvent<HTMLSelectElement>) => void;
}
```

### 2.4 Form Submission Flow

```
1. User fills form
2. Client-side validation
3. Turnstile CAPTCHA verification
4. Submit Step 1 (user profile) → Get submissionId
5. Phone verification modal → Submit Step 2 (phone validation)
6. Submit Step 3 (datacenter) → Final submission
7. Success/Warning screen
```

**Key Features**:
- Bootstrap endpoint for geolocation data (async, non-blocking)
- Multi-step API submission with IDs passed between steps
- Cloudflare Turnstile for bot protection
- Phone verification via SMS/Call
- GDPR consent handling
- Campaign ID (campid) tracking
- Error handling at field and form levels

### 2.5 Internationalization (i18n)

```typescript
// src/context/LanguageContext.tsx
const { t, currentLanguage } = useLanguage();

// Usage
t('common.fields.firstName', 'First name')
t('trial.steps.step1.banner', 'Step 1: User Profile')
```

Translation files in `src/locales/`:
- en.json
- de.json
- es.json
- fr.json
- ja.json
- pt-br.json

### 2.6 Current Limitations

1. **New Form = New Route**: Each form requires new page components
2. **Hardcoded Field Groups**: Field combinations are hardcoded in step/form components
3. **Limited Reusability**: Trial-specific logic prevents use for other form types
4. **Configuration Sprawl**: Product configs are TypeScript files requiring rebuilds
5. **Duplicate Code**: Similar forms duplicate validation, submission, and layout logic
6. **No Visual Form Builder**: Non-technical users cannot create forms
7. **Heavy Client Bundle**: All form logic loaded even if not used

### 2.7 Current Strengths (To Preserve)

1. ✅ **Type Safety**: Full TypeScript coverage
2. ✅ **Reusable Field Components**: Well-abstracted input/select/field components
3. ✅ **Validation Architecture**: Client + server validation with error display
4. ✅ **Accessibility**: Proper ARIA labels, keyboard navigation
5. ✅ **Performance**: Server-side rendering, code splitting
6. ✅ **Internationalization**: Full i18n support
7. ✅ **Analytics Integration**: Form events tracked via GTM
8. ✅ **Security**: Turnstile CAPTCHA, input sanitization
9. ✅ **Error Handling**: Graceful degradation, retry logic

---

## 3. Target Architecture

### 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Form Definition Layer                      │
│  (YAML Configs + Markdown Content)                           │
│  • Schema validation                                          │
│  • Static analysis                                            │
│  • Version control                                            │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Form Builder Engine                         │
│  • Parse YAML → JSON                                          │
│  • Validate against schema                                    │
│  • Generate TypeScript types (build-time)                     │
│  • Create form metadata registry                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Form Renderer (Runtime)                     │
│  • Route matching: /forms/:formId                             │
│  • Load form config from registry                             │
│  • Render flat HTML (SSR)                                     │
│  • Hydrate with React (progressive enhancement)               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              Reusable Component Library                       │
│  (Existing components from current system)                    │
│  • FormField, InputField, SelectField                         │
│  • Button, Card, ErrorAlert                                   │
│  • PhoneInput, VerificationCodeInput                          │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Routing Strategy

**Current**: Hard-coded routes
```
/trial/remote-support
/trial/remote-support/single
/trial/remote-support/iframe
/trial/privileged-remote-access
/trial/privileged-remote-access/single
```

**Proposed**: Dynamic routing with single catch-all route
```
/forms/[formId]/[[...variant]]

Examples:
/forms/remote-support-trial
/forms/remote-support-trial/single
/forms/remote-support-trial/iframe
/forms/contact-sales
/forms/demo-request
/forms/event-registration/webinar-2025-q1
```

### 3.3 Directory Structure (Proposed)

```
src/
├── forms/                                    # NEW: Form definitions
│   ├── schemas/
│   │   └── form-schema.json                  # JSON Schema for YAML validation
│   ├── definitions/                          # YAML form configs
│   │   ├── trial/
│   │   │   ├── remote-support.yaml
│   │   │   └── privileged-remote-access.yaml
│   │   ├── marketing/
│   │   │   ├── contact-sales.yaml
│   │   │   ├── demo-request.yaml
│   │   │   └── newsletter-signup.yaml
│   │   └── events/
│   │       └── webinar-registration.yaml
│   ├── content/                              # Markdown content files
│   │   ├── legal/
│   │   │   ├── privacy-notice.md
│   │   │   └── terms-of-service.md
│   │   └── instructions/
│   │       └── phone-verification.md
│   └── generated/                            # Build-time generated files
│       ├── form-registry.json                # All forms metadata
│       └── form-types.ts                     # TypeScript types
├── pages/
│   └── forms/
│       └── [formId]/
│           └── [[...variant]].tsx            # Dynamic route handler
├── lib/
│   └── forms/                                # NEW: Form builder engine
│       ├── parser/
│       │   ├── yaml-parser.ts                # Parse YAML to JSON
│       │   ├── schema-validator.ts           # Validate against schema
│       │   └── type-generator.ts             # Generate TS types
│       ├── renderer/
│       │   ├── FormRenderer.tsx              # Main form rendering component
│       │   ├── FieldRenderer.tsx             # Render individual fields
│       │   ├── SectionRenderer.tsx           # Render form sections
│       │   └── LayoutRenderer.tsx            # Render layouts (single/multi-step)
│       ├── registry/
│       │   ├── form-registry.ts              # Form lookup/retrieval
│       │   └── component-registry.ts         # Map field types to components
│       ├── validation/
│       │   ├── field-validator.ts            # Validate individual fields
│       │   ├── form-validator.ts             # Validate entire form
│       │   └── rules.ts                      # Validation rule definitions
│       ├── submission/
│       │   ├── form-submitter.ts             # Handle form submission
│       │   ├── api-adapter.ts                # Adapt to different APIs
│       │   └── retry-logic.ts                # Submission retry strategy
│       └── hooks/
│           ├── useFormConfig.ts              # Load form configuration
│           ├── useFormState.ts               # Manage form state
│           ├── useFormSubmission.ts          # Handle submission
│           └── useFieldValidation.ts         # Validate fields
├── components/forms/                         # Existing reusable components
│   ├── FormField.tsx                         # Keep as-is
│   ├── InputField.tsx                        # Keep as-is
│   ├── SelectField.tsx                       # Keep as-is
│   └── ...
└── components/ui/                            # Existing UI components
    └── ...
```

### 3.4 Key Architectural Principles

1. **Configuration over Code**: Form structure defined in YAML, not TypeScript
2. **Build-Time Optimization**: Generate types, validate schemas, create registry at build time
3. **Runtime Efficiency**: Minimal JavaScript, server-side rendering, progressive enhancement
4. **Component Reuse**: Leverage existing component library
5. **Type Safety**: Generated TypeScript types from YAML schemas
6. **Backward Compatibility**: Existing trial forms continue to work during migration
7. **Developer Experience**: Hot reload YAML changes in dev mode
8. **Content Management**: Markdown for rich text content, versioned alongside forms

---

## 4. YAML Schema Design

### 4.1 Form Configuration Schema

```yaml
# forms/definitions/trial/remote-support.yaml

# Metadata
id: remote-support-trial
version: 1.0.0
name: Remote Support Trial Request
description: Trial request form for BeyondTrust Remote Support product

# SEO & Metadata
metadata:
  title: "BeyondTrust Remote Support • Free Trial"
  description: "Get secure, lightning-fast remote support with BeyondTrust. Start your free trial today."
  keywords:
    - remote support
    - trial
    - beyondtrust
  canonical: "/forms/remote-support-trial"

# Internationalization
i18n:
  enabled: true
  defaultLocale: en
  supportedLocales:
    - en
    - de
    - es
    - fr
    - ja
    - pt-br

# Layout Configuration
layout:
  mode: unified  # 'stepped' or 'unified'
  variant: default  # default, modal, iframe
  theme: light  # light, dark, auto
  width: 6xl  # max-width class

  # For stepped forms
  steps:
    - id: user-profile
      title: "trial.steps.step1.banner"
      description: "trial.steps.step1.description"
      sections:
        - user-info
        - company-info
    - id: phone-validation
      title: "trial.steps.step2.banner"
      description: "trial.steps.step2.description"
      sections:
        - phone-verification
    - id: datacenter-selection
      title: "trial.steps.step3.banner"
      description: "trial.steps.step3.description"
      sections:
        - datacenter

# Form Sections
sections:
  - id: user-info
    title: null  # No section title
    layout: grid  # grid, stack
    columns: 2  # for grid layout
    fields:
      - firstName
      - lastName
      - emailAddress
      - jobRole

  - id: company-info
    title: null
    layout: grid
    columns: 2
    fields:
      - companyName
      - industry

  - id: phone-verification
    title: null
    layout: stack
    fields:
      - phoneNumber

  - id: datacenter
    title: null
    layout: stack
    fields:
      - datacenter

# Field Definitions
fields:
  firstName:
    id: firstName
    type: text
    component: InputField
    label: "common.fields.firstName"
    placeholder: ""
    required: true
    validation:
      minLength: 2
      maxLength: 50
      pattern: "^[a-zA-Z\\s\\-']+$"
      errorMessages:
        required: "First name is required"
        minLength: "First name must be at least 2 characters"
        maxLength: "First name must be less than 50 characters"
        pattern: "First name can only contain letters, spaces, hyphens, and apostrophes"
    attributes:
      autoComplete: given-name
      spellCheck: false

  lastName:
    id: lastName
    type: text
    component: InputField
    label: "common.fields.lastName"
    required: true
    validation:
      minLength: 2
      maxLength: 50
      pattern: "^[a-zA-Z\\s\\-']+$"
      errorMessages:
        required: "Last name is required"
        minLength: "Last name must be at least 2 characters"
        maxLength: "Last name must be less than 50 characters"
        pattern: "Last name can only contain letters, spaces, hyphens, and apostrophes"
    attributes:
      autoComplete: family-name

  emailAddress:
    id: emailAddress
    type: email
    component: InputField
    label: "common.fields.email"
    required: true
    validation:
      maxLength: 100
      pattern: "^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$"
      errorMessages:
        required: "Email is required"
        maxLength: "Email must be less than 100 characters"
        pattern: "Please enter a valid email address"
    attributes:
      autoComplete: email
    features:
      preload: true  # Can be pre-populated from bootstrap

  jobRole:
    id: jobRole
    type: select
    component: SelectField
    label: "common.fields.jobRole"
    placeholder: "common.actions.selectRole"
    required: true
    options:
      - value: "C-Level"
        labelKey: "common.jobRoles.cLevel"
      - value: "Vice President"
        labelKey: "common.jobRoles.vicePresident"
      - value: "Director"
        labelKey: "common.jobRoles.director"
      - value: "Manager"
        labelKey: "common.jobRoles.manager"
      - value: "Administrator"
        labelKey: "common.jobRoles.administrator"
      - value: "Engineer/Architect"
        labelKey: "common.jobRoles.engineerArchitect"
    validation:
      errorMessages:
        required: "Job role is required"

  companyName:
    id: companyName
    type: text
    component: InputField
    label: "common.fields.companyName"
    required: true
    validation:
      minLength: 2
      maxLength: 100
      errorMessages:
        required: "Company name is required"
        minLength: "Company name must be at least 2 characters"
        maxLength: "Company name must be less than 100 characters"
    attributes:
      autoComplete: organization

  industry:
    id: industry
    type: select
    component: SelectField
    label: "common.fields.industry"
    placeholder: "common.actions.selectIndustry"
    required: false
    options:
      - value: "aerospace-defense"
        labelKey: "common.industries.aerospaceDefense"
      - value: "agriculture"
        labelKey: "common.industries.agriculture"
      # ... (full list of 30+ industries)
      - value: "other"
        labelKey: "common.industries.other"

  phoneNumber:
    id: phoneNumber
    type: phone
    component: PhoneInput  # Custom international phone input
    label: "common.fields.phone"
    required: true
    validation:
      errorMessages:
        required: "Phone number is required"
        invalid: "Please enter a valid phone number"
    features:
      verification: true  # Trigger phone verification flow
      verificationMethods:
        - sms
        - call
      initialCountry: auto  # Use geolocation

  datacenter:
    id: datacenter
    type: select
    component: SelectField
    label: "common.actions.selectRegion"
    placeholder: "common.actions.selectRegion"
    required: true
    options:
      - value: "AP_SE"
        labelKey: "trial.datacenters.apSingapore"
      - value: "AU_S"
        labelKey: "trial.datacenters.auSydney"
      - value: "CA_C"
        labelKey: "trial.datacenters.caCentral"
      - value: "EU_DE"
        labelKey: "trial.datacenters.euGermany"
      - value: "EU_UK"
        labelKey: "trial.datacenters.euLondon"
      - value: "EU_FR"
        labelKey: "trial.datacenters.euParis"
      - value: "IN_W"
        labelKey: "trial.datacenters.indiaWest"
      - value: "AF_S"
        labelKey: "trial.datacenters.southAfrica"
      - value: "BR_E"
        labelKey: "trial.datacenters.southAmericaEast"
      - value: "ME_C"
        labelKey: "trial.datacenters.uae"
      - value: "US_C"
        labelKey: "trial.datacenters.usCentral"
      - value: "US_E"
        labelKey: "trial.datacenters.usEast"
      - value: "US_W"
        labelKey: "trial.datacenters.usWest"
    features:
      autoSelect: geolocation  # Auto-select based on user location

# Hidden Fields (for tracking/metadata)
hiddenFields:
  - id: formType
    value: "remote-support"
  - id: product
    value: "Remote Support"
  - id: campid
    source: query  # Read from URL query param
  - id: landingPageUrl
    source: location  # Current page URL
  - id: submissionId
    source: api  # Populated from API response
  - id: referenceId
    source: api

# Security
security:
  turnstile:
    enabled: true
    siteKey: "${NEXT_PUBLIC_TURNSTILE_SITE_KEY}"
    position: beforeSubmit  # Show before or after form fields
  rateLimit:
    enabled: true
    maxAttempts: 5
    windowMinutes: 60

# GDPR Compliance
gdpr:
  enabled: true
  checkGeolocation: true  # Only show for GDPR regions
  consentType: checkbox  # checkbox, implicit
  consentField:
    id: emailOptIn
    label: "trial.steps.step1.emailOptInLabel"
    required: false
    defaultValue: false
  legalText:
    content: "trial.steps.step1.privacyText"
    links:
      - text: "trial.steps.step1.privacyPolicy"
        url: "https://www.beyondtrust.com/privacy-notice"
      - text: "trial.steps.step1.managePreferencesLink"
        url: "https://www.beyondtrust.com/forms/manage-subscriptions"

# Submission Configuration
submission:
  api:
    endpoint: "${API_BASE_URL}/api/trials/submit"
    method: POST
    mode: multi-step  # multi-step, single
    steps:
      - stepNumber: 1
        fields:
          - firstName
          - lastName
          - emailAddress
          - jobRole
          - companyName
          - industry
          - turnstileToken
          - emailOptIn
          - campid
          - landingPageUrl
        response:
          submissionId: true  # Expect submissionId in response
          referenceId: true
      - stepNumber: 2
        fields:
          - phoneNumber
          - verificationCode
          - verificationMethod
          - submissionId
        response:
          continueToNextStep: true
      - stepNumber: 3
        fields:
          - datacenter
          - submissionId
          - referenceId
        response:
          countryBlocked: true  # Check for restrictions
          builderStatus: true
    headers:
      Content-Type: application/json
    timeout: 20000
    retry:
      maxAttempts: 3
      backoff: exponential

# Success/Error Handling
outcomes:
  success:
    component: SuccessStep
    message: "trial.success.message"
    actions:
      - type: redirect
        url: null  # Stay on page
      - type: analytics
        event: trial_request_complete
        properties:
          product: "Remote Support"
          datacenter: "${datacenter}"

  warning:
    component: WarningStep
    conditions:
      - field: countryBlocked
        value: true
        reason: blocked-country
      - field: countryType
        value: unknown
        reason: unknown-country
      - field: builderStatus
        value: unavailable
        reason: api-unavailable
    message: "trial.warning.message"
    actions:
      - type: analytics
        event: trial_request_warning

  error:
    component: ErrorAlert
    message: "trial.error.message"
    retry: true

# Analytics
analytics:
  provider: gtm
  events:
    - trigger: stepComplete
      event: trial_step_complete
      properties:
        step: "${step}"
        formMode: "${layout.mode}"
    - trigger: fieldError
      event: trial_field_error
      properties:
        field: "${field}"
        error: "${error}"
    - trigger: submit
      event: trial_form_submit
      properties:
        product: "Remote Support"

# Features
features:
  bootstrap:
    enabled: true
    endpoint: "${API_BASE_URL}/api/bootstrap"
    params:
      - campid
      - em
    async: true  # Non-blocking
  geolocation:
    enabled: true
    autoSelectDatacenter: true
  phoneVerification:
    enabled: true
    modal: true
    methods:
      - sms
      - call
  darkMode:
    enabled: true
    default: auto

# A/B Testing (future consideration)
experiments: []
```

### 4.2 JSON Schema for Validation

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Form Configuration Schema",
  "type": "object",
  "required": ["id", "version", "name", "fields", "submission"],
  "properties": {
    "id": {
      "type": "string",
      "pattern": "^[a-z0-9-]+$",
      "description": "Unique form identifier (URL-safe)"
    },
    "version": {
      "type": "string",
      "pattern": "^\\d+\\.\\d+\\.\\d+$",
      "description": "Semantic version"
    },
    "name": {
      "type": "string",
      "description": "Human-readable form name"
    },
    "description": {
      "type": "string"
    },
    "metadata": {
      "type": "object",
      "properties": {
        "title": { "type": "string" },
        "description": { "type": "string" },
        "keywords": {
          "type": "array",
          "items": { "type": "string" }
        },
        "canonical": { "type": "string" }
      }
    },
    "layout": {
      "type": "object",
      "required": ["mode"],
      "properties": {
        "mode": {
          "type": "string",
          "enum": ["stepped", "unified"]
        },
        "variant": {
          "type": "string",
          "enum": ["default", "modal", "iframe"]
        },
        "theme": {
          "type": "string",
          "enum": ["light", "dark", "auto"]
        }
      }
    },
    "fields": {
      "type": "object",
      "additionalProperties": {
        "$ref": "#/definitions/field"
      }
    },
    "submission": {
      "type": "object",
      "required": ["api"],
      "properties": {
        "api": {
          "type": "object",
          "required": ["endpoint", "method"],
          "properties": {
            "endpoint": { "type": "string" },
            "method": {
              "type": "string",
              "enum": ["GET", "POST", "PUT", "PATCH"]
            }
          }
        }
      }
    }
  },
  "definitions": {
    "field": {
      "type": "object",
      "required": ["id", "type", "component", "label"],
      "properties": {
        "id": { "type": "string" },
        "type": {
          "type": "string",
          "enum": ["text", "email", "phone", "select", "checkbox", "textarea"]
        },
        "component": { "type": "string" },
        "label": { "type": "string" },
        "required": { "type": "boolean" },
        "validation": {
          "type": "object",
          "properties": {
            "minLength": { "type": "number" },
            "maxLength": { "type": "number" },
            "pattern": { "type": "string" }
          }
        }
      }
    }
  }
}
```

### 4.3 Simplified Form Example (Contact Form)

```yaml
# forms/definitions/marketing/contact-sales.yaml

id: contact-sales
version: 1.0.0
name: Contact Sales
description: General sales inquiry form

layout:
  mode: unified
  variant: default
  width: 4xl

sections:
  - id: contact-info
    layout: grid
    columns: 2
    fields:
      - firstName
      - lastName
      - emailAddress
      - phoneNumber
      - companyName
      - message

fields:
  firstName:
    id: firstName
    type: text
    component: InputField
    label: "common.fields.firstName"
    required: true
    validation:
      minLength: 2
      maxLength: 50

  lastName:
    id: lastName
    type: text
    component: InputField
    label: "common.fields.lastName"
    required: true
    validation:
      minLength: 2
      maxLength: 50

  emailAddress:
    id: emailAddress
    type: email
    component: InputField
    label: "common.fields.email"
    required: true
    validation:
      pattern: "^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$"

  phoneNumber:
    id: phoneNumber
    type: phone
    component: PhoneInput
    label: "common.fields.phone"
    required: false

  companyName:
    id: companyName
    type: text
    component: InputField
    label: "common.fields.companyName"
    required: true

  message:
    id: message
    type: textarea
    component: TextAreaField
    label: "common.fields.message"
    required: true
    attributes:
      rows: 5
      maxLength: 1000

submission:
  api:
    endpoint: "${API_BASE_URL}/api/contact/submit"
    method: POST
    mode: single

outcomes:
  success:
    message: "contact.success.message"
    actions:
      - type: redirect
        url: "/thank-you"

security:
  turnstile:
    enabled: true

gdpr:
  enabled: true
```

---

## 5. Form Rendering Engine

### 5.1 Component Architecture

```typescript
// lib/forms/renderer/FormRenderer.tsx

import { FormConfig } from '@/lib/forms/types';

interface FormRendererProps {
  formId: string;
  variant?: 'default' | 'modal' | 'iframe';
}

export const FormRenderer: React.FC<FormRendererProps> = ({ formId, variant }) => {
  // 1. Load form configuration from registry
  const formConfig = useFormConfig(formId);

  // 2. Initialize form state
  const formState = useFormState(formConfig);

  // 3. Setup validation
  const validation = useFieldValidation(formConfig);

  // 4. Setup submission handler
  const submission = useFormSubmission(formConfig);

  // 5. Render based on layout mode
  if (formConfig.layout.mode === 'stepped') {
    return <SteppedFormRenderer config={formConfig} />;
  }

  return <UnifiedFormRenderer config={formConfig} />;
};
```

### 5.2 Field Rendering Logic

```typescript
// lib/forms/renderer/FieldRenderer.tsx

import { FieldConfig } from '@/lib/forms/types';
import { componentRegistry } from '@/lib/forms/registry/component-registry';

interface FieldRendererProps {
  fieldConfig: FieldConfig;
  value: any;
  error?: string;
  onChange: (value: any) => void;
}

export const FieldRenderer: React.FC<FieldRendererProps> = ({
  fieldConfig,
  value,
  error,
  onChange
}) => {
  // Get component from registry based on field type
  const Component = componentRegistry.get(fieldConfig.component);

  if (!Component) {
    console.error(`Component not found: ${fieldConfig.component}`);
    return null;
  }

  // Translate label using i18n
  const { t } = useLanguage();
  const label = t(fieldConfig.label, fieldConfig.label);

  return (
    <FormField
      id={fieldConfig.id}
      label={label}
      required={fieldConfig.required}
      error={error}
    >
      <Component
        id={fieldConfig.id}
        value={value}
        onChange={onChange}
        {...fieldConfig.attributes}
      />
    </FormField>
  );
};
```

### 5.3 Component Registry

```typescript
// lib/forms/registry/component-registry.ts

import InputField from '@/components/forms/InputField';
import SelectField from '@/components/forms/SelectField';
import PhoneInput from '@/components/forms/PhoneInputWithButtons';
import TextAreaField from '@/components/forms/TextAreaField';
// ... import other components

type ComponentMap = Map<string, React.ComponentType<any>>;

class ComponentRegistry {
  private components: ComponentMap = new Map();

  constructor() {
    // Register default components
    this.register('InputField', InputField);
    this.register('SelectField', SelectField);
    this.register('PhoneInput', PhoneInput);
    this.register('TextAreaField', TextAreaField);
    // ... register other components
  }

  register(name: string, component: React.ComponentType<any>) {
    this.components.set(name, component);
  }

  get(name: string): React.ComponentType<any> | undefined {
    return this.components.get(name);
  }
}

export const componentRegistry = new ComponentRegistry();
```

### 5.4 Form Registry

```typescript
// lib/forms/registry/form-registry.ts

import { FormConfig } from '@/lib/forms/types';
import formRegistryData from '@/forms/generated/form-registry.json';

class FormRegistry {
  private forms: Map<string, FormConfig> = new Map();

  constructor() {
    // Load all forms from generated registry
    Object.entries(formRegistryData).forEach(([id, config]) => {
      this.forms.set(id, config as FormConfig);
    });
  }

  get(formId: string): FormConfig | undefined {
    return this.forms.get(formId);
  }

  getAll(): FormConfig[] {
    return Array.from(this.forms.values());
  }

  exists(formId: string): boolean {
    return this.forms.has(formId);
  }
}

export const formRegistry = new FormRegistry();
```

### 5.5 Build-Time Processing

```typescript
// scripts/build-forms.ts

import fs from 'fs';
import path from 'path';
import yaml from 'js-yaml';
import Ajv from 'ajv';

const FORMS_DIR = path.join(process.cwd(), 'src/forms/definitions');
const SCHEMA_PATH = path.join(process.cwd(), 'src/forms/schemas/form-schema.json');
const OUTPUT_DIR = path.join(process.cwd(), 'src/forms/generated');

async function buildForms() {
  console.log('Building form registry...');

  // 1. Load schema
  const schema = JSON.parse(fs.readFileSync(SCHEMA_PATH, 'utf-8'));
  const ajv = new Ajv({ allErrors: true });
  const validate = ajv.compile(schema);

  // 2. Find all YAML files
  const formFiles = findYamlFiles(FORMS_DIR);

  const registry = {};
  const errors = [];

  // 3. Process each form
  for (const file of formFiles) {
    try {
      const content = fs.readFileSync(file, 'utf-8');
      const formConfig = yaml.load(content);

      // Validate against schema
      const valid = validate(formConfig);
      if (!valid) {
        errors.push({
          file,
          errors: validate.errors
        });
        continue;
      }

      // Add to registry
      registry[formConfig.id] = formConfig;

      console.log(`✓ Processed: ${formConfig.id}`);
    } catch (error) {
      errors.push({ file, error: error.message });
    }
  }

  // 4. Write registry
  if (!fs.existsSync(OUTPUT_DIR)) {
    fs.mkdirSync(OUTPUT_DIR, { recursive: true });
  }

  fs.writeFileSync(
    path.join(OUTPUT_DIR, 'form-registry.json'),
    JSON.stringify(registry, null, 2)
  );

  // 5. Generate TypeScript types
  generateTypes(registry);

  // 6. Report results
  console.log(`\nBuilt ${Object.keys(registry).length} forms`);
  if (errors.length > 0) {
    console.error(`\n${errors.length} errors found:`);
    errors.forEach(err => console.error(err));
    process.exit(1);
  }
}

function findYamlFiles(dir: string): string[] {
  // Recursively find all .yaml files
  // Implementation details...
}

function generateTypes(registry: any) {
  // Generate TypeScript types from registry
  // Implementation details...
}

buildForms();
```

### 5.6 Dynamic Route Handler

```typescript
// pages/forms/[formId]/[[...variant]].tsx

import { GetStaticPaths, GetStaticProps } from 'next';
import { FormRenderer } from '@/lib/forms/renderer/FormRenderer';
import { formRegistry } from '@/lib/forms/registry/form-registry';

interface FormPageProps {
  formId: string;
  variant?: string;
}

export default function FormPage({ formId, variant }: FormPageProps) {
  return (
    <FormRenderer
      formId={formId}
      variant={variant as any}
    />
  );
}

export const getStaticPaths: GetStaticPaths = async () => {
  const forms = formRegistry.getAll();

  const paths = forms.flatMap(form => [
    { params: { formId: form.id, variant: [] } },
    { params: { formId: form.id, variant: ['single'] } },
    { params: { formId: form.id, variant: ['iframe'] } }
  ]);

  return {
    paths,
    fallback: false
  };
};

export const getStaticProps: GetStaticProps = async ({ params }) => {
  const formId = params?.formId as string;
  const variant = params?.variant?.[0] || 'default';

  // Verify form exists
  if (!formRegistry.exists(formId)) {
    return { notFound: true };
  }

  return {
    props: {
      formId,
      variant
    }
  };
};
```

---

## 6. Migration Strategy

### 6.1 Phased Approach

**Phase 1: Foundation (Weeks 1-2)**
- ✅ Create YAML schema definition
- ✅ Build schema validator
- ✅ Create form registry structure
- ✅ Build YAML parser
- ✅ Generate TypeScript types from YAML

**Phase 2: Rendering Engine (Weeks 3-4)**
- ✅ Build FormRenderer component
- ✅ Build FieldRenderer component
- ✅ Create component registry
- ✅ Implement validation engine
- ✅ Implement submission handler

**Phase 3: Migration of Trial Forms (Weeks 5-6)**
- ✅ Convert remote-support form to YAML
- ✅ Convert privileged-remote-access form to YAML
- ✅ Test parity with existing forms
- ✅ A/B test both implementations
- ✅ Full cutover to YAML-based forms

**Phase 4: New Form Types (Weeks 7-8)**
- ✅ Create contact-sales form
- ✅ Create demo-request form
- ✅ Create newsletter-signup form
- ✅ Create event-registration form

**Phase 5: Optimization & Documentation (Weeks 9-10)**
- ✅ Performance optimization
- ✅ Developer documentation
- ✅ Content editor guide
- ✅ Monitoring & analytics

### 6.2 Backward Compatibility

During migration, both systems will coexist:

```
Old routes (continue working):
/trial/remote-support → TrialForm.tsx
/trial/remote-support/single → UnifiedTrialForm.tsx

New routes (gradually introduced):
/forms/remote-support-trial → FormRenderer (YAML-based)
/forms/remote-support-trial/single → FormRenderer (YAML-based)
```

Redirect strategy:
1. Phase 1-2: Both routes work, old is canonical
2. Phase 3: Both routes work, new is canonical
3. Phase 4: Old routes redirect to new routes
4. Phase 5: Remove old route handlers

### 6.3 Testing Strategy

**Unit Tests**:
- YAML parser
- Schema validator
- Field validators
- Component registry
- Form registry

**Integration Tests**:
- Full form rendering
- Field validation
- Form submission
- Error handling
- Success/warning flows

**E2E Tests** (Cypress):
- Complete form workflows
- Multi-step forms
- Phone verification
- GDPR consent
- Geolocation features

**Performance Tests**:
- Lighthouse scores
- Core Web Vitals
- Bundle size analysis
- Server response times

**A/B Testing**:
- Conversion rate comparison
- User experience metrics
- Error rate comparison
- Time-to-completion

---

## 7. Technical Specifications

### 7.1 Type Definitions

```typescript
// lib/forms/types.ts

export interface FormConfig {
  id: string;
  version: string;
  name: string;
  description?: string;
  metadata?: FormMetadata;
  i18n?: I18nConfig;
  layout: LayoutConfig;
  sections?: SectionConfig[];
  fields: Record<string, FieldConfig>;
  hiddenFields?: HiddenFieldConfig[];
  security?: SecurityConfig;
  gdpr?: GDPRConfig;
  submission: SubmissionConfig;
  outcomes: OutcomesConfig;
  analytics?: AnalyticsConfig;
  features?: FeaturesConfig;
}

export interface FormMetadata {
  title: string;
  description: string;
  keywords?: string[];
  canonical?: string;
}

export interface LayoutConfig {
  mode: 'stepped' | 'unified';
  variant?: 'default' | 'modal' | 'iframe';
  theme?: 'light' | 'dark' | 'auto';
  width?: string;
  steps?: StepConfig[];
}

export interface StepConfig {
  id: string;
  title: string;
  description?: string;
  sections: string[];
}

export interface SectionConfig {
  id: string;
  title?: string;
  layout: 'grid' | 'stack';
  columns?: number;
  fields: string[];
}

export interface FieldConfig {
  id: string;
  type: 'text' | 'email' | 'phone' | 'select' | 'checkbox' | 'textarea';
  component: string;
  label: string;
  placeholder?: string;
  required: boolean;
  validation?: ValidationConfig;
  attributes?: Record<string, any>;
  options?: OptionConfig[];
  features?: FieldFeatures;
}

export interface ValidationConfig {
  minLength?: number;
  maxLength?: number;
  pattern?: string;
  errorMessages?: Record<string, string>;
}

export interface OptionConfig {
  value: string;
  labelKey: string;
}

export interface FieldFeatures {
  preload?: boolean;
  verification?: boolean;
  autoSelect?: 'geolocation' | 'none';
}

export interface SubmissionConfig {
  api: APIConfig;
}

export interface APIConfig {
  endpoint: string;
  method: 'GET' | 'POST' | 'PUT' | 'PATCH';
  mode: 'single' | 'multi-step';
  steps?: StepSubmissionConfig[];
  headers?: Record<string, string>;
  timeout?: number;
  retry?: RetryConfig;
}

export interface OutcomesConfig {
  success: OutcomeConfig;
  warning?: OutcomeConfig;
  error?: OutcomeConfig;
}

export interface OutcomeConfig {
  component?: string;
  message: string;
  actions?: ActionConfig[];
}

// ... additional types
```

### 7.2 Validation Rules Engine

```typescript
// lib/forms/validation/field-validator.ts

import { FieldConfig, ValidationConfig } from '@/lib/forms/types';

export class FieldValidator {
  validate(value: any, fieldConfig: FieldConfig): string | null {
    if (fieldConfig.required && !value) {
      return this.getErrorMessage('required', fieldConfig);
    }

    const validation = fieldConfig.validation;
    if (!validation) return null;

    // String validations
    if (typeof value === 'string') {
      if (validation.minLength && value.length < validation.minLength) {
        return this.getErrorMessage('minLength', fieldConfig);
      }

      if (validation.maxLength && value.length > validation.maxLength) {
        return this.getErrorMessage('maxLength', fieldConfig);
      }

      if (validation.pattern) {
        const regex = new RegExp(validation.pattern);
        if (!regex.test(value)) {
          return this.getErrorMessage('pattern', fieldConfig);
        }
      }
    }

    return null;
  }

  private getErrorMessage(type: string, fieldConfig: FieldConfig): string {
    return fieldConfig.validation?.errorMessages?.[type] || `Validation failed: ${type}`;
  }
}

export const fieldValidator = new FieldValidator();
```

### 7.3 Form State Management

```typescript
// lib/forms/hooks/useFormState.ts

import { useState, useCallback } from 'react';
import { FormConfig } from '@/lib/forms/types';

export function useFormState(config: FormConfig) {
  // Initialize state with default values
  const [values, setValues] = useState<Record<string, any>>(() => {
    const initial: Record<string, any> = {};
    Object.keys(config.fields).forEach(fieldId => {
      initial[fieldId] = '';
    });
    return initial;
  });

  const [errors, setErrors] = useState<Record<string, string>>({});
  const [touched, setTouched] = useState<Record<string, boolean>>({});

  const setValue = useCallback((fieldId: string, value: any) => {
    setValues(prev => ({ ...prev, [fieldId]: value }));
  }, []);

  const setError = useCallback((fieldId: string, error: string) => {
    setErrors(prev => ({ ...prev, [fieldId]: error }));
  }, []);

  const clearError = useCallback((fieldId: string) => {
    setErrors(prev => {
      const next = { ...prev };
      delete next[fieldId];
      return next;
    });
  }, []);

  const setTouched = useCallback((fieldId: string) => {
    setTouched(prev => ({ ...prev, [fieldId]: true }));
  }, []);

  const reset = useCallback(() => {
    const initial: Record<string, any> = {};
    Object.keys(config.fields).forEach(fieldId => {
      initial[fieldId] = '';
    });
    setValues(initial);
    setErrors({});
    setTouched({});
  }, [config]);

  return {
    values,
    errors,
    touched,
    setValue,
    setError,
    clearError,
    setTouched,
    reset
  };
}
```

### 7.4 Submission Handler

```typescript
// lib/forms/hooks/useFormSubmission.ts

import { useState, useCallback } from 'react';
import { FormConfig } from '@/lib/forms/types';
import { apiClient } from '@/lib/forms/submission/api-client';

export function useFormSubmission(config: FormConfig) {
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [submissionError, setSubmissionError] = useState<string | null>(null);

  const submit = useCallback(async (values: Record<string, any>) => {
    setIsSubmitting(true);
    setSubmissionError(null);

    try {
      const response = await apiClient.submit(
        config.submission.api.endpoint,
        values,
        config.submission.api
      );

      if (response.success) {
        return { success: true, data: response.data };
      } else {
        setSubmissionError(response.message || 'Submission failed');
        return { success: false, errors: response.errors };
      }
    } catch (error) {
      const message = error instanceof Error ? error.message : 'Unknown error';
      setSubmissionError(message);
      return { success: false, error: message };
    } finally {
      setIsSubmitting(false);
    }
  }, [config]);

  return {
    submit,
    isSubmitting,
    submissionError
  };
}
```

---

## 8. Implementation Plan

### 8.1 Week-by-Week Breakdown

**Week 1: Schema & Infrastructure**
- [ ] Define JSON Schema for form configuration
- [ ] Create YAML example for remote-support trial
- [ ] Build YAML parser utility
- [ ] Build schema validator
- [ ] Create form registry data structure
- [ ] Setup build script to process YAML files

**Week 2: Type Generation & Registry**
- [ ] Build TypeScript type generator from YAML
- [ ] Create component registry
- [ ] Create form registry
- [ ] Build form lookup utilities
- [ ] Write unit tests for parsers and validators

**Week 3: Rendering Engine - Core**
- [ ] Build FormRenderer component
- [ ] Build FieldRenderer component
- [ ] Build SectionRenderer component
- [ ] Implement component mapping logic
- [ ] Add error boundaries

**Week 4: Rendering Engine - Features**
- [ ] Implement validation engine
- [ ] Build form state management hooks
- [ ] Build submission handler
- [ ] Add i18n support
- [ ] Add GDPR logic

**Week 5: Trial Form Migration Part 1**
- [ ] Convert remote-support.yaml
- [ ] Test remote-support rendering
- [ ] Test remote-support validation
- [ ] Test remote-support submission
- [ ] Fix bugs and edge cases

**Week 6: Trial Form Migration Part 2**
- [ ] Convert privileged-remote-access.yaml
- [ ] Add phone verification support
- [ ] Add Turnstile integration
- [ ] Add geolocation support
- [ ] Full E2E testing

**Week 7: New Forms**
- [ ] Create contact-sales.yaml
- [ ] Create demo-request.yaml
- [ ] Create newsletter-signup.yaml
- [ ] Test all new forms
- [ ] Analytics integration

**Week 8: Polish & Optimization**
- [ ] Performance optimization
- [ ] Bundle size reduction
- [ ] Accessibility audit
- [ ] Cross-browser testing
- [ ] Load testing

**Week 9: Documentation**
- [ ] Developer documentation
- [ ] Content editor guide
- [ ] YAML schema reference
- [ ] Component API docs
- [ ] Migration guide

**Week 10: Launch & Monitor**
- [ ] Gradual rollout (10% → 50% → 100%)
- [ ] Monitor error rates
- [ ] Monitor performance metrics
- [ ] Monitor conversion rates
- [ ] Decommission old code

### 8.2 Success Criteria

**Performance**:
- ✅ Lighthouse score ≥ 90
- ✅ First Contentful Paint < 1.5s
- ✅ Time to Interactive < 3.0s
- ✅ Total bundle size increase < 20KB

**Functionality**:
- ✅ 100% feature parity with existing trial forms
- ✅ All E2E tests passing
- ✅ Zero critical bugs in production
- ✅ Conversion rate within 5% of baseline

**Developer Experience**:
- ✅ Create new form in < 30 minutes
- ✅ YAML validation in IDE (JSON Schema)
- ✅ Hot reload working in dev mode
- ✅ Clear error messages for misconfigurations

**Scalability**:
- ✅ Support 10+ forms without performance degradation
- ✅ Build time < 60 seconds
- ✅ Type checking for all form configs
- ✅ Reusable components for all field types

---

## 9. Risk Assessment

### 9.1 Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| YAML parsing errors cause runtime failures | High | Medium | Build-time validation, schema checks, unit tests |
| Type safety lost with dynamic config | Medium | Medium | Generate TS types at build time, strict validation |
| Performance degradation from abstraction | Medium | Low | Benchmark early, optimize render paths, SSR |
| Component registry mismatches | High | Low | Runtime checks, dev-mode warnings, type guards |
| Breaking changes to existing forms | High | Low | Maintain old routes during migration, A/B test |
| Complex forms don't fit YAML model | Medium | Medium | Iterative schema evolution, escape hatches |

### 9.2 Business Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Conversion rate drops | High | Low | A/B testing, gradual rollout, quick rollback |
| Development takes longer than planned | Medium | Medium | Phased approach, MVP first, iterate |
| Non-technical users struggle with YAML | Medium | Medium | Visual form builder (future), templates, docs |
| Maintenance burden increases | Low | Low | Clear documentation, automated testing |

### 9.3 Rollback Plan

If critical issues arise post-launch:

1. **Immediate Rollback**: Revert to old route handlers (< 5 minutes)
2. **Partial Rollback**: Route specific forms back to old implementation
3. **Fix Forward**: Deploy hotfix if issue is isolated
4. **Data Integrity**: No data loss as APIs remain unchanged

---

## 10. Appendices

### 10.1 Glossary

- **Form Config**: YAML file defining form structure and behavior
- **Form Registry**: Build-time generated index of all forms
- **Component Registry**: Runtime mapping of field types to React components
- **Form Renderer**: Dynamic component that renders forms from config
- **Field Renderer**: Component that renders individual form fields
- **Stepped Form**: Multi-step wizard-style form
- **Unified Form**: Single-page form with all fields visible

### 10.2 References

- [Next.js Documentation](https://nextjs.org/docs)
- [JSON Schema Specification](https://json-schema.org/)
- [YAML Specification](https://yaml.org/spec/)
- [React Hook Form](https://react-hook-form.com/) (reference, not dependency)

### 10.3 Example File Structure

```
src/forms/
├── definitions/
│   ├── trial/
│   │   ├── remote-support.yaml
│   │   └── privileged-remote-access.yaml
│   ├── marketing/
│   │   ├── contact-sales.yaml
│   │   ├── demo-request.yaml
│   │   └── newsletter-signup.yaml
│   └── events/
│       └── webinar-2025-q1.yaml
├── schemas/
│   └── form-schema.json
├── content/
│   └── legal/
│       └── privacy-notice.md
└── generated/
    ├── form-registry.json
    └── form-types.ts
```

### 10.4 Code Examples Repository Structure

For detailed code examples to implement each section, refer to:
- FormRenderer implementation: `lib/forms/renderer/FormRenderer.tsx`
- Field validation: `lib/forms/validation/field-validator.ts`
- YAML parser: `lib/forms/parser/yaml-parser.ts`
- Build script: `scripts/build-forms.ts`

### 10.5 Open Questions

1. Should we support custom JavaScript validators in YAML configs?
2. How do we handle complex conditional field logic?
3. Should we build a visual form builder UI?
4. What's the strategy for form versioning and migration?
5. How do we handle form-specific analytics events?

---

## Document Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-10-21 | Claude | Initial LLD creation |

---

**Next Steps**:
1. Review this LLD with engineering team
2. Get approval from stakeholders
3. Refine YAML schema based on feedback
4. Begin Phase 1 implementation
5. Set up project tracking in Jira/Linear

