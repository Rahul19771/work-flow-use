# OpenDental Integration - Complete Implementation Guide

## Overview

This document provides a comprehensive guide for the OpenDental integration implementation in the Aspire Healthcare platform. The integration enables seamless communication between our system and OpenDental practice management software, supporting patient management, appointment scheduling, provider synchronization, and clinical workflows.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites & Setup](#prerequisites--setup)
3. [Core Components](#core-components)
4. [API Integration](#api-integration)
5. [Data Models](#data-models)
6. [Authentication & Security](#authentication--security)
7. [Feature Implementation](#feature-implementation)
8. [Integration Points](#integration-points)
9. [Error Handling](#error-handling)
10. [Testing & Validation](#testing--validation)
11. [Deployment Guide](#deployment-guide)
12. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

### System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Aspire Healthcare Platform                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ API Layer   │  │ Data Layer  │  │ Serverless  │             │
│  │             │  │             │  │ Functions   │             │
│  │ - Visit     │  │ - Entities  │  │             │             │
│  │ - CallLogs  │  │ - Services  │  │ - Tasks     │             │
│  │ - Patients  │  │ - Models    │  │ - Webhooks  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                    OpenDental Client                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ Patient     │  │ Appointment │  │ Provider    │             │
│  │ Management  │  │ Scheduling  │  │ Sync        │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                     OpenDental API                              │
│                  (api.opendental.com)                           │
└─────────────────────────────────────────────────────────────────┘
```

### Component Distribution

| Layer | Component | Purpose |
|-------|-----------|---------|
| **API Layer** | `api/src/logic/visit.ts` | Visit management and slot retrieval |
| **API Layer** | `api/src/logic/calllogs.ts` | Task processing and appointment actions |
| **Data Layer** | `data/src/srv/opendental.ts` | Core OpenDental client implementation |
| **Data Layer** | `data/src/ent/visit.ts` | Visit data entity management |
| **Data Layer** | `data/src/ent/provider.ts` | Provider data entity management |
| **Data Layer** | `data/src/ent/calllogs.ts` | Patient context generation |
| **Serverless** | `serverless/functions/src/logic/task.ts` | Task processing workflows |

---

## Prerequisites & Setup

### OpenDental API Requirements

1. **API Access**: OpenDental Cloud subscription with API access enabled
2. **API Keys**: Developer Key (`devKey`) and Customer Key (`custKey`)
3. **Permissions**: Administrative access to configure API endpoints
4. **Network**: Whitelisted IP addresses for API calls

### Environment Configuration

```bash
# AWS Secrets Manager
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_REGION=us-west-2

# OpenDental API Endpoint
OPENDENTAL_API_BASE_URL=https://api.opendental.com/api/v1
```

### Database Setup

```sql
-- Practice configuration
UPDATE practices SET 
  ehr = 'OPEN_DENTAL',
  openDentalIntegration = {
    "openDentalApiKeySecretName": "practice-{id}-opendental-api-key",
    "clinicNumber": 1,
    "operatoryIdToProviderIdMap": {},
    "operatoryTypeToIdsMap": {},
    "operatoryTypeToProviderIdsMap": {}
  }
WHERE id = 'practice-id';
```

---

## Core Components

### 1. OpenDental Client (`data/src/srv/opendental.ts`)

The main service class that handles all OpenDental API interactions.

#### Key Features:
- **Authentication**: ODFHIR token-based authentication
- **Rate Limiting**: Built-in 1-second delays for API compliance
- **Error Handling**: Comprehensive error logging and recovery
- **Data Formatting**: Automatic data transformation between systems

#### Core Methods:

```typescript
class OpenDentalClient {
  // Patient Operations
  async getPatients(params: GetPatientsParams): Promise<OpenDentalPatient[]>
  async createPatient(params: CreatePatientParams): Promise<OpenDentalPatient>
  async getPatientById(params: GetPatientParams): Promise<OpenDentalPatient>
  
  // Appointment Operations
  async getAppointments(params: GetAppointmentsParams): Promise<OpenDentalAppointment[]>
  async createAppointment(params: CreateAppointmentParams): Promise<OpenDentalAppointment>
  async updateAppointment(params: UpdateAppointmentParams): Promise<OpenDentalAppointment>
  async breakAppointment(params: BreakAppointmentParams): Promise<OpenDentalAppointment>
  async confirmAppointment(params: ConfirmAppointmentParams): Promise<OpenDentalAppointment>
  
  // Provider Operations
  async getProviders(params: GetProvidersParams): Promise<OpenDentalProvider[]>
  
  // Operatory Operations
  async getOperatories(params: GetOperatoriesParams): Promise<OpenDentalOperatory[]>
  async checkOperatoryAvailability(params: CheckAvailabilityParams): Promise<AvailabilityResult>
  
  // Slot Operations
  async getAppointmentSlots(params: GetSlotsParams): Promise<OpenDentalAppointmentSlot[]>
  
  // Sync Operations
  async syncPatients(params: SyncParams): Promise<number>
  async syncProviders(params: SyncParams): Promise<number>
  async syncAppointments(params: SyncParams): Promise<number>
}
```


### 2. Visit Management (`api/src/logic/visit.ts`)

Handles visit-related operations and slot management.

#### Key Functions:

```typescript
// Get available appointment slots
const _getVisitSlotsFromOpenDental = async (params: {
  practice: Practice;
  serviceId: string;
  providerExternalIds?: string[];
}) => Promise<ProviderAvailability[]>

// Get slots for rescheduling specific visits
const _getVisitSlotsByVisitIdFromOpenDental = async (params: {
  practice: Practice;
  visitId: string;
}) => Promise<ProviderAvailability[]>

// Get patient visit history
const getPatientVisitsFromOpenDental = async (params: {
  patientExternalId: string;
  practiceId: string;
  practiceTimezone?: string;
  services: Service[];
  providers: Provider[];
}) => Promise<PatientVisitSummary>
```

### 3. Task Processing (`api/src/logic/calllogs.ts`)

Processes automated tasks for appointment management.

#### Supported Task Types:

```typescript
enum TaskActionType {
  CANCEL_APPOINTMENT = "CANCEL_APPOINTMENT",
  RESCHEDULE_APPOINTMENT = "RESCHEDULE_APPOINTMENT", 
  CREATE_APPOINTMENT = "CREATE_APPOINTMENT",
  ONBOARD_NEW_PATIENT = "ONBOARD_NEW_PATIENT"
}
```

#### Task Processing Flow:

```typescript
const handleOpenDentalTask = async (task: Task, practice: Practice) => {
  const openDentalClient = await initOpenDentalClient(practice.objectID);
  
  switch (task.actionDetails.type) {
    case TaskActionType.CANCEL_APPOINTMENT:
      await openDentalClient.breakAppointment({
        appointmentId: task.actionDetails.externalAppointmentId,
        sendToUnscheduledList: true
      });
      break;
      
    case TaskActionType.CREATE_APPOINTMENT:
      await openDentalClient.createAppointment({
        patientId: task.actionDetails.externalPatientId,
        startTime: task.actionDetails.appointmentTimeInISO,
        operatoryId: "1", // Default operatory
        providerId: task.actionDetails.providerExternalId
      });
      break;
      
    // ... other task types
  }
}
```

---

## API Integration

### Authentication

OpenDental uses ODFHIR (OpenDental FHIR) authentication:

```typescript
// Request Headers
{
  "Authorization": `ODFHIR ${devKey}/${custKey}`,
  "Content-Type": "application/json",
  "Accept": "application/json"
}
```

### Base URL Structure

```
https://api.opendental.com/api/v1/{endpoint}
```

### Key Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/patients` | GET | Retrieve patients |
| `/patients` | POST | Create new patient |
| `/patients/{id}` | GET | Get patient by ID |
| `/appointments` | GET | Retrieve appointments |
| `/appointments` | POST | Create appointment |
| `/appointments/{id}` | PUT | Update appointment |
| `/appointments/{id}/Break` | PUT | Cancel appointment |
| `/appointments/{id}/Confirm` | PUT | Confirm appointment |
| `/appointments/Slots` | GET | Get available slots |
| `/providers` | GET | Retrieve providers |
| `/operatories` | GET | Retrieve operatories |

### Rate Limiting

- **Limit**: 1 request per second
- **Implementation**: Built-in 1-second delays in sync operations
- **Retry Logic**: Exponential backoff for failed requests

---

## Data Models

### OpenDental Patient

```typescript
interface OpenDentalPatient {
  PatNum: number;           // Primary key
  FName: string;            // First name
  LName: string;            // Last name
  MiddleI?: string;         // Middle initial
  Email?: string;           // Email address
  WirelessPhone?: string;   // Mobile phone
  HmPhone?: string;         // Home phone
  WkPhone?: string;         // Work phone
  Birthdate?: string;       // YYYY-MM-DD format
  Address?: string;         // Street address
  Address2?: string;        // Address line 2
  City?: string;            // City
  State?: string;           // State
  Zip?: string;             // ZIP code
  SSN?: string;             // Social Security Number
  Gender?: OpenDentalGender;
  PatStatus?: OpenDentalPatientStatus;
  Position?: OpenDentalPosition;
  PriProv?: number;         // Primary provider ID
  SecProv?: number;         // Secondary provider ID
  ChartNumber?: string;     // Chart number
  ClinicNum?: number;       // Clinic number
  DateFirstVisit?: string;  // First visit date
  PreferContactMethod?: OpenDentalPatientPreferContactMethod;
  PreferRecallMethod?: OpenDentalPatientPreferContactMethod;
  PreferConfirmMethod?: OpenDentalPatientPreferContactMethod;
}
```

### OpenDental Appointment

```typescript
interface OpenDentalAppointment {
  AptNum: number;           // Primary key
  PatNum: number;           // Patient ID
  ProvNum?: number;         // Provider ID
  ProvHyg?: number;         // Hygienist ID
  Op: number;               // Operatory ID
  AptDateTime: string;      // Appointment date/time
  Pattern: string;          // Time pattern (X = 5 min blocks)
  AptStatus: OpenDentalAppointmentStatus;
  Note?: string;            // Appointment notes
  Confirmed?: number;       // Confirmation status
  IsNewPatient?: boolean;   // New patient flag
  ProcDescript?: string;    // Procedure description
  ClinicNum?: number;       // Clinic number
  AptDateTimeArrived?: string;    // Arrival time
  AptDateTimeSeated?: string;     // Seated time
  AptDateTimeDismissed?: string;  // Dismissed time
}
```

### OpenDental Provider

```typescript
interface OpenDentalProvider {
  ProvNum: number;          // Primary key
  FName: string;            // First name
  LName: string;            // Last name
  Abbr: string;             // Abbreviation
  NationalProvID?: string;  // NPI number
  // Additional provider fields...
}
```

### OpenDental Operatory

```typescript
interface OpenDentalOperatory {
  OperatoryNum: number;     // Primary key
  OpName: string;           // Operatory name
  Abbrev: string;           // Abbreviation
  ProvDentist?: number;     // Default dentist
  ProvHygienist?: number;   // Default hygienist
  ClinicNum?: number;       // Clinic number
  IsActive: boolean;        // Active status
}
```


---

## Authentication & Security

### AWS Secrets Manager Integration

```typescript
// Secret Structure
{
  "devKey": "your-developer-key",
  "custKey": "your-customer-key"
}

// Retrieval Pattern
const getOpenDentalCredentials = async (practiceId: string) => {
  const client = new SecretsManagerClient({ region: "us-west-2" });
  const practice = await Practice.fetch({ id: practiceId });
  
  const secret = await client.send(
    new GetSecretValueCommand({
      SecretId: practice.openDentalIntegration.openDentalApiKeySecretName
    })
  );
  
  return JSON.parse(secret.SecretString);
};
```

### Security Best Practices

1. **Credential Storage**: All API keys stored in AWS Secrets Manager
2. **Network Security**: API calls made from whitelisted servers
3. **Data Encryption**: TLS 1.2+ for all API communications
4. **Access Control**: Role-based access to OpenDental credentials
5. **Audit Logging**: All API calls logged for security monitoring

---

## Feature Implementation

### 1. Patient Management

#### Patient Creation
```typescript
const createPatient = async (patientData: CreatePatientParams) => {
  const openDentalClient = await initOpenDentalClient(practiceId);
  
  try {
    const patient = await openDentalClient.createPatient({
      firstName: patientData.firstName,
      lastName: patientData.lastName,
      dateOfBirth: patientData.dateOfBirth,
      phoneNumber: patientData.phoneNumber,
      email: patientData.email
    });
    
    return patient;
  } catch (error) {
    if (error.message.includes("already exists")) {
      // Handle duplicate patient
      const existingPatient = await findExistingPatient(patientData);
      return existingPatient;
    }
    throw error;
  }
};
```

#### Patient Synchronization
```typescript
const syncPatients = async (practiceId: string) => {
  const openDentalClient = await initOpenDentalClient(practiceId);
  
  let offset = 0;
  const BATCH_SIZE = 500;
  let totalSynced = 0;
  
  while (true) {
    const patients = await openDentalClient.getPatients({
      limit: BATCH_SIZE,
      offset: offset
    });
    
    if (patients.length === 0) break;
    
    const syncedCount = await bulkUpsertPatients(patients, practiceId);
    totalSynced += syncedCount;
    offset += patients.length;
    
    // Rate limiting
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
  
  return totalSynced;
};
```

### 2. Appointment Management

#### Appointment Creation
```typescript
const createAppointment = async (appointmentData: CreateAppointmentParams) => {
  const openDentalClient = await initOpenDentalClient(practiceId);
  
  // Check operatory availability
  const availabilityCheck = await openDentalClient.checkOperatoryAvailability({
    operatoryId: appointmentData.operatoryId,
    appointmentTime: appointmentData.startTime,
    durationMinutes: appointmentData.duration
  });
  
  if (!availabilityCheck.available) {
    throw new Error(`Operatory not available: ${availabilityCheck.reason}`);
  }
  
  const appointment = await openDentalClient.createAppointment({
    patientId: appointmentData.patientId,
    providerId: appointmentData.providerId,
    startTime: appointmentData.startTime,
    operatoryId: appointmentData.operatoryId,
    notes: appointmentData.notes
  });
  
  return appointment;
};
```

#### Appointment Rescheduling
```typescript
const rescheduleAppointment = async (rescheduleData: RescheduleParams) => {
  const openDentalClient = await initOpenDentalClient(practiceId);
  
  // Get current appointment
  const currentAppointment = await openDentalClient.getAppointmentById({
    appointmentId: rescheduleData.appointmentId
  });
  
  // Break current appointment
  await openDentalClient.breakAppointment({
    appointmentId: rescheduleData.appointmentId,
    sendToUnscheduledList: false
  });
  
  // Create new appointment
  const newAppointment = await openDentalClient.createAppointment({
    patientId: currentAppointment.PatNum.toString(),
    providerId: currentAppointment.ProvNum?.toString(),
    startTime: rescheduleData.newStartTime,
    operatoryId: currentAppointment.Op.toString(),
    notes: "Rescheduled appointment"
  });
  
  return newAppointment;
};
```

### 3. Slot Management

#### Available Slots Retrieval
```typescript
const getAvailableSlots = async (slotParams: GetSlotsParams) => {
  const openDentalClient = await initOpenDentalClient(practiceId);
  
  const slots = await openDentalClient.getAppointmentSlots({
    dateStart: slotParams.startDate,
    dateEnd: slotParams.endDate,
    lengthMinutes: slotParams.duration,
    ProvNum: slotParams.providerId,
    Op: slotParams.operatoryId
  });
  
  // Group slots by provider and day
  const groupedSlots = groupSlotsByProviderAndDay(slots, practiceTimezone);
  
  return groupedSlots;
};
```

### 4. Provider Synchronization

#### Provider Data Sync
```typescript
const syncProviders = async (practiceId: string) => {
  const openDentalClient = await initOpenDentalClient(practiceId);
  
  const providers = await openDentalClient.getProviders({
    clinicNumber: practice.openDentalIntegration.clinicNumber
  });
  
  const formattedProviders = providers.map(provider => ({
    externalId: provider.ProvNum.toString(),
    name: `${provider.FName} ${provider.LName}`.trim() || provider.Abbr,
    firstName: provider.FName,
    lastName: provider.LName,
    npi: provider.NationalProvID || "",
    abbreviatedName: provider.Abbr
  }));
  
  const syncedCount = await bulkUpsertProviders(formattedProviders, practiceId);
  return syncedCount;
};
```

---

## Integration Points

### 1. Call Log Processing

The system integrates with call logs to automatically process appointment-related tasks:

```typescript
// api/src/logic/calllogs.ts
const performTaskByActionType = async (params: { task: Task }) => {
  const practice = await Practice.fetch({ id: params.task.practiceId });
  
  switch (practice.ehr) {
    case "OPEN_DENTAL":
      await handleOpenDentalTask(params.task, practice);
      break;
    // ... other EHR systems
  }
};
```

### 2. Patient Context Generation

Generate contextual patient information for AI assistants:

```typescript
// data/src/ent/calllogs.ts
const _generatePatientContextFromOpenDental = async (params: {
  patient: Patient;
  practice: Practice;
}) => {
  const providers = await getExternalProviders(params.practice.objectID);
  const services = await getServices(params.practice.objectID);
  const visitSummary = await getPatientVisitSummary(params.patient.objectID);
  
  return `
    # PATIENT DATA
    - Name: ${params.patient.firstName} ${params.patient.lastName}
    - Email: ${params.patient.email}
    - Phone: ${params.patient.phoneNumber}
    
    # PROVIDER DATA
    ${providers.map(p => `- ${p.name} (ID: ${p.externalId})`).join('\n')}
    
    # UPCOMING APPOINTMENTS
    ${visitSummary.patientUpcomingVisits.map(v => 
      `- ${v.AppointmentStartDateAndTimeFormatted} with ${v.PractitionerName}`
    ).join('\n')}
  `;
};
```

### 3. Visit Management

Handle visit-related operations across different EHR systems:

```typescript
// api/src/logic/visit.ts
const getVisitSlotsByVisitId = async (params: {
  practiceId: string;
  visitId: string;
}) => {
  const practice = await Practice.fetch({ id: params.practiceId });
  
  switch (practice.ehr) {
    case "OPEN_DENTAL":
      return _getVisitSlotsByVisitIdFromOpenDental({ ...params, practice });
    // ... other EHR systems
  }
};
```

---

## Error Handling

### 1. API Error Handling

```typescript
class OpenDentalClient {
  private async handleApiError(error: any, operation: string) {
    console.error(`OpenDental API Error - ${operation}:`, error);
    
    if (error.response?.status === 401) {
      throw new Error("Authentication failed - check API credentials");
    }
    
    if (error.response?.status === 429) {
      // Rate limit exceeded
      await this.waitForRateLimit();
      throw new Error("Rate limit exceeded - please retry");
    }
    
    if (error.response?.status === 400) {
      const errorMessage = error.response?.data?.error || error.message;
      throw new Error(`Bad request: ${errorMessage}`);
    }
    
    throw new Error(`OpenDental API error: ${error.message}`);
  }
}
```

### 2. Data Validation

```typescript
const validatePatientData = (patientData: CreatePatientParams) => {
  if (!patientData.firstName || !patientData.lastName) {
    throw new Error("Patient first name and last name are required");
  }
  
  if (patientData.email && !isValidEmail(patientData.email)) {
    throw new Error("Invalid email format");
  }
  
  if (patientData.phoneNumber && !isValidPhoneNumber(patientData.phoneNumber)) {
    throw new Error("Invalid phone number format");
  }
};
```

### 3. Retry Logic

```typescript
const retryWithBackoff = async (
  operation: () => Promise<any>,
  maxRetries: number = 3
) => {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (attempt === maxRetries) {
        throw error;
      }
      
      const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
};
```


---

## Testing & Validation

### 1. Unit Tests

```typescript
// tests/opendental.test.ts
describe('OpenDental Client', () => {
  let client: OpenDentalClient;
  
  beforeEach(() => {
    client = new OpenDentalClient('test-dev-key', 'test-cust-key');
  });
  
  describe('Patient Management', () => {
    it('should create a new patient', async () => {
      const patientData = {
        firstName: 'John',
        lastName: 'Doe',
        email: 'john@example.com',
        phoneNumber: '+1234567890'
      };
      
      const patient = await client.createPatient(patientData);
      
      expect(patient.PatNum).toBeDefined();
      expect(patient.FName).toBe('John');
      expect(patient.LName).toBe('Doe');
    });
  });
  
  describe('Appointment Management', () => {
    it('should create an appointment', async () => {
      const appointmentData = {
        patientId: '123',
        providerId: '456',
        startTime: '2024-01-15T10:00:00Z',
        operatoryId: '1'
      };
      
      const appointment = await client.createAppointment(appointmentData);
      
      expect(appointment.AptNum).toBeDefined();
      expect(appointment.PatNum).toBe(123);
    });
  });
});
```

### 2. Integration Tests

```typescript
// tests/integration/opendental-integration.test.ts
describe('OpenDental Integration', () => {
  it('should sync patients from OpenDental', async () => {
    const practiceId = 'test-practice-id';
    const syncedCount = await syncPatients(practiceId);
    
    expect(syncedCount).toBeGreaterThan(0);
  });
  
  it('should handle appointment lifecycle', async () => {
    // Create patient
    const patient = await createPatient(testPatientData);
    
    // Create appointment
    const appointment = await createAppointment({
      patientId: patient.PatNum.toString(),
      startTime: '2024-01-15T10:00:00Z',
      operatoryId: '1'
    });
    
    // Reschedule appointment
    const rescheduledAppointment = await rescheduleAppointment({
      appointmentId: appointment.AptNum.toString(),
      newStartTime: '2024-01-15T14:00:00Z'
    });
    
    // Cancel appointment
    await cancelAppointment({
      appointmentId: rescheduledAppointment.AptNum.toString()
    });
    
    expect(rescheduledAppointment.AptDateTime).toContain('14:00');
  });
});
```

### 3. API Validation Tests

```typescript
// tests/api/opendental-api.test.ts
describe('OpenDental API Validation', () => {
  it('should handle authentication correctly', async () => {
    const client = new OpenDentalClient('invalid-key', 'invalid-key');
    
    await expect(client.getPatients()).rejects.toThrow('Authentication failed');
  });
  
  it('should respect rate limits', async () => {
    const client = new OpenDentalClient(validDevKey, validCustKey);
    
    // Make multiple rapid requests
    const requests = Array(10).fill(null).map(() => client.getPatients());
    
    const startTime = Date.now();
    await Promise.all(requests);
    const duration = Date.now() - startTime;
    
    // Should take at least 9 seconds due to rate limiting
    expect(duration).toBeGreaterThan(9000);
  });
});
```

---

## Deployment Guide

### 1. Environment Setup

```bash
# Production Environment Variables
export NODE_ENV=production
export AWS_REGION=us-west-2
export OPENDENTAL_API_BASE_URL=https://api.opendental.com/api/v1

# Development Environment
export NODE_ENV=development
export AWS_REGION=us-west-2
export OPENDENTAL_API_BASE_URL=https://api.opendental.com/api/v1
```

### 2. AWS Secrets Manager Setup

```bash
# Create secret for each practice
aws secretsmanager create-secret \
  --name "practice-${PRACTICE_ID}-opendental-api-key" \
  --description "OpenDental API credentials for practice ${PRACTICE_ID}" \
  --secret-string '{"devKey":"your-dev-key","custKey":"your-cust-key"}'
```

### 3. Database Migration

```sql
-- Add OpenDental configuration to practices
ALTER TABLE practices ADD COLUMN openDentalIntegration JSON;

-- Update existing practices
UPDATE practices 
SET openDentalIntegration = JSON_OBJECT(
  'openDentalApiKeySecretName', CONCAT('practice-', id, '-opendental-api-key'),
  'clinicNumber', 1,
  'operatoryIdToProviderIdMap', JSON_OBJECT(),
  'operatoryTypeToIdsMap', JSON_OBJECT(),
  'operatoryTypeToProviderIdsMap', JSON_OBJECT()
)
WHERE ehr = 'OPEN_DENTAL';
```

### 4. Build and Deploy

```bash
# Build all packages
cd data && yarn build
cd ../api && yarn build
cd ../serverless/functions && yarn build

# Deploy serverless functions
serverless deploy --stage production

# Deploy API services
docker build -t aspire-api .
docker push your-registry/aspire-api:latest
kubectl apply -f k8s/api-deployment.yaml
```

### 5. Health Checks

```typescript
// health-check.ts
export const openDentalHealthCheck = async (practiceId: string) => {
  try {
    const client = await initOpenDentalClient(practiceId);
    const providers = await client.getProviders({ limit: 1 });
    
    return {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      providerCount: providers.length
    };
  } catch (error) {
    return {
      status: 'unhealthy',
      timestamp: new Date().toISOString(),
      error: error.message
    };
  }
};
```

---

## Troubleshooting

### Common Issues

#### 1. Authentication Failures

**Problem**: `Authentication failed - check API credentials`

**Solutions**:
- Verify dev key and customer key in AWS Secrets Manager
- Check if API keys are active in OpenDental
- Ensure practice has API access enabled

```bash
# Check secret exists
aws secretsmanager describe-secret --secret-id practice-123-opendental-api-key

# Test API credentials
curl -X GET "https://api.opendental.com/api/v1/providers" \
  -H "Authorization: ODFHIR your-dev-key/your-cust-key" \
  -H "Content-Type: application/json"
```

#### 2. Rate Limiting

**Problem**: `Rate limit exceeded - please retry`

**Solutions**:
- Implement exponential backoff
- Add delays between API calls
- Use batch processing for bulk operations

```typescript
// Rate-limited sync implementation
const syncWithRateLimit = async (syncFunction: () => Promise<any>) => {
  const delay = 1000; // 1 second
  
  try {
    const result = await syncFunction();
    await new Promise(resolve => setTimeout(resolve, delay));
    return result;
  } catch (error) {
    if (error.message.includes('rate limit')) {
      await new Promise(resolve => setTimeout(resolve, delay * 2));
      return syncWithRateLimit(syncFunction);
    }
    throw error;
  }
};
```

#### 3. Data Synchronization Issues

**Problem**: Patient/appointment data not syncing properly

**Solutions**:
- Check date format consistency (YYYY-MM-DD HH:mm:ss)
- Verify clinic number configuration
- Ensure proper timezone handling

```typescript
// Debug sync issues
const debugSync = async (practiceId: string) => {
  const client = await initOpenDentalClient(practiceId);
  
  console.log('Testing API connection...');
  const providers = await client.getProviders({ limit: 1 });
  console.log(`Found ${providers.length} providers`);
  
  console.log('Testing patient sync...');
  const patients = await client.getPatients({ limit: 5 });
  console.log(`Found ${patients.length} patients`);
  
  console.log('Testing appointment sync...');
  const appointments = await client.getAppointments({
    dateUpdatedStartInPracticeTz: dayjs().format('YYYY-MM-DD')
  });
  console.log(`Found ${appointments.length} appointments`);
};
```

#### 4. Operatory Availability Issues

**Problem**: "No available operatory found" errors

**Solutions**:
- Verify operatory configuration in OpenDental
- Check operatory-to-provider mappings
- Ensure operatory is active and available

```typescript
// Debug operatory availability
const debugOperatoryAvailability = async (practiceId: string) => {
  const client = await initOpenDentalClient(practiceId);
  
  const operatories = await client.getOperatories();
  console.log('Available operatories:', operatories);
  
  for (const operatory of operatories) {
    const availability = await client.checkOperatoryAvailability({
      operatoryId: operatory.OperatoryNum.toString(),
      appointmentTime: dayjs().add(1, 'day').toISOString(),
      durationMinutes: 60
    });
    
    console.log(`Operatory ${operatory.OpName}:`, availability);
  }
};
```

---

## Conclusion

This comprehensive guide covers the complete OpenDental integration implementation. The system provides:

- **Complete CRUD operations** for patients, appointments, and providers
- **Real-time synchronization** with OpenDental practice management system
- **Automated task processing** for appointment management
- **Robust error handling** and retry mechanisms
- **Performance optimizations** for high-volume operations
- **Comprehensive monitoring** and logging capabilities

The integration follows industry best practices for:
- Security (AWS Secrets Manager, TLS encryption)
- Scalability (batch processing, connection pooling)
- Reliability (retry logic, error handling)
- Maintainability (modular architecture, comprehensive testing)

For additional support or feature requests, please refer to the development team or create an issue in the project repository.

