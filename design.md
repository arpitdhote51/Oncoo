# Design Document

## Overview

Oncoo is a personal health management platform built on AWS serverless architecture, designed to provide a secure, private space for cancer patients to manage their treatment routines, diet plans, and medical reports with controlled access by their treatment team. The platform focuses on individual patient care and treatment tracking.

The platform serves patients, survivors, caregivers, family members, and friends across India and beyond, with particular focus on rural accessibility through offline capabilities and multi-language support. The architecture emphasizes security, scalability, and cost-effectiveness through serverless technologies.

## Architecture

### High-Level Architecture

The system follows a serverless, event-driven architecture on AWS:

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Mobile App    │    │    Web PWA       │    │   Admin Panel   │
│  (React Native) │    │    (React)       │    │    (React)      │
└─────────┬───────┘    └────────┬─────────┘    └─────────┬───────┘
          │                     │                        │
          └─────────────────────┼────────────────────────┘
                                │
                    ┌───────────▼────────────┐
                    │     API Gateway        │
                    │   (REST + WebSocket)   │
                    └───────────┬────────────┘
                                │
                    ┌───────────▼────────────┐
                    │    Lambda Functions    │
                    │  (Node.js/TypeScript)  │
                    └───────────┬────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
┌───────▼────────┐    ┌────────▼────────┐    ┌────────▼────────┐
│   DynamoDB     │    │   S3 Buckets    │    │   Bedrock AI    │
│ (User Data &   │    │ (Media Files &  │    │ (Embeddings &   │
│  Community)    │    │  Static Assets) │    │  Translation)   │
└────────────────┘    └─────────────────┘    └─────────────────┘
```

### Security Architecture

- **Authentication**: AWS Cognito with MFA support
- **Authorization**: JWT tokens with role-based access control
- **Encryption**: KMS for data at rest, TLS 1.3 for data in transit
- **Privacy**: Row-level security implemented in application layer
- **Compliance**: GDPR/DPDP compliance with audit logging

### Offline-First Design

- **Local Storage**: SQLite for offline data caching
- **Sync Strategy**: Event-driven synchronization when connectivity restored
- **Conflict Resolution**: Last-write-wins with user notification for conflicts
- **Progressive Web App**: Service workers for offline functionality

## Components and Interfaces

### Core Components

#### 1. User Management Service
- **Responsibilities**: Authentication, profile management, privacy controls
- **Interfaces**: 
  - `POST /auth/signup` - Waitlist registration
  - `POST /auth/login` - User authentication
  - `PUT /users/{id}/profile` - Profile updates
  - `PUT /users/{id}/privacy` - Privacy settings
  - `DELETE /users/{id}` - Account deletion (GDPR compliance)

#### 2. Community Service
- **Responsibilities**: Groups, posts, chronological feeds
- **Interfaces**:
  - `GET /communities` - List available communities
  - `POST /communities/{id}/posts` - Create new post
  - `GET /communities/{id}/feed` - Chronological feed
  - `PUT /posts/{id}` - Update post
  - `DELETE /posts/{id}` - Delete post

#### 3. Routine Planning Service
- **Responsibilities**: Daily routines, medication schedules, chemotherapy cycles, appointment tracking
- **Interfaces**:
  - `POST /routines` - Create routine plan
  - `PUT /routines/{id}` - Update routine
  - `GET /routines/{userId}` - Get user routines
  - `POST /routines/{id}/complete` - Mark routine task complete
  - `GET /routines/{userId}/calendar` - Get routine calendar view
  - `POST /chemotherapy/cycles` - Create chemotherapy cycle
  - `PUT /chemotherapy/cycles/{id}` - Update cycle status and details
  - `GET /chemotherapy/{userId}/cycles` - Get patient's chemo cycles
  - `POST /chemotherapy/cycles/{id}/side-effects` - Log side effects

#### 4. Diet Planning Service
- **Responsibilities**: Nutrition plans, dietary recommendations, meal tracking
- **Interfaces**:
  - `POST /diet-plans` - Create diet plan
  - `PUT /diet-plans/{id}` - Update diet plan
  - `GET /diet-plans/{userId}` - Get user diet plans
  - `POST /meals` - Log meal intake
  - `GET /nutrition/{userId}/analysis` - Get nutrition analysis

#### 5. Medical Report Service
- **Responsibilities**: Report storage, analysis, doctor access control
- **Interfaces**:
  - `POST /reports/upload` - Upload medical report
  - `GET /reports/{userId}` - Get user reports (patient access)
  - `POST /reports/{id}/grant-access` - Doctor grants access to analysis
  - `GET /reports/{id}/analysis` - Get AI analysis (if authorized)
  - `PUT /reports/{id}/doctor-notes` - Add doctor notes

#### 6. AI Analysis Service
- **Responsibilities**: Medical report analysis, treatment insights, translation
- **Interfaces**:
  - `POST /ai/analyze-report` - Analyze medical report
  - `POST /ai/translate` - Translate content
  - `GET /ai/insights/{reportId}` - Get treatment insights
  - `POST /ai/risk-assessment` - Generate risk assessment

#### 8. Doctor Dashboard Service
- **Responsibilities**: Patient overview, progress tracking, treatment monitoring
- **Interfaces**:
  - `GET /doctor/patients` - Get list of assigned patients
  - `GET /doctor/patient/{id}/overview` - Get comprehensive patient overview
  - `GET /doctor/patient/{id}/progress` - Get treatment progress analytics
  - `PUT /doctor/patient/{id}/notes` - Add doctor notes and observations
  - `POST /doctor/patient/{id}/alerts` - Set up progress alerts and thresholds
  - `GET /doctor/patient/{id}/timeline` - Get patient treatment timeline
  - `PUT /doctor/patient/{id}/treatment-plan` - Update treatment plan

#### 9. Notification Service
- **Responsibilities**: Medication reminders, appointment alerts, routine notifications
- **Interfaces**:
  - `POST /notifications/send` - Send notification
  - `GET /notifications/{userId}` - User notifications
  - `PUT /notifications/{id}/read` - Mark as read
  - `POST /notifications/schedule` - Schedule routine reminders

## Data Models

### User Model
```typescript
interface User {
  id: string;
  email: string;
  phone?: string;
  profile: {
    name: string;
    journeyStage: 'newly_diagnosed' | 'in_treatment' | 'survivor' | 'caregiver';
    cancerType?: string;
    language: string;
    photo?: string;
    dateOfBirth: Date;
    emergencyContact: {
      name: string;
      phone: string;
      relationship: string;
    };
  };
  privacy: {
    profileVisibility: 'doctor_only' | 'private';
    allowDoctorAccess: boolean;
    shareAnalysisWithDoctor: boolean;
  };
  preferences: {
    notifications: NotificationPreferences;
    accessibility: AccessibilitySettings;
    reminderSettings: ReminderSettings;
  };
  assignedDoctors: string[]; // Doctor IDs with access
  createdAt: Date;
  updatedAt: Date;
  lastActiveAt: Date;
}
```

### Routine Model
```typescript
interface Routine {
  id: string;
  userId: string;
  name: string;
  description: string;
  type: 'medication' | 'chemotherapy' | 'exercise' | 'appointment' | 'therapy' | 'custom';
  schedule: {
    frequency: 'daily' | 'weekly' | 'monthly' | 'cycle_based' | 'custom';
    times: string[]; // Time of day in HH:MM format
    days?: number[]; // Days of week (0-6) for weekly
    dates?: number[]; // Days of month for monthly
    customPattern?: string; // Cron-like pattern for custom
  };
  medication?: {
    name: string;
    dosage: string;
    instructions: string;
    route: 'oral' | 'iv' | 'injection' | 'topical' | 'other';
  };
  chemotherapy?: {
    regimenName: string; // e.g., "FOLFOX", "AC-T", "Carboplatin-Paclitaxel"
    cycleNumber: number; // Current cycle (e.g., 3)
    totalCycles: number; // Total planned cycles (e.g., 6)
    treatmentPhase: 'pre_medication' | 'infusion' | 'post_medication' | 'recovery';
    drugs: {
      name: string;
      dosage: string;
      infusionDuration?: string; // e.g., "2 hours"
      sequence: number; // Order of administration
    }[];
    cycleLength: number; // Days between cycles (e.g., 21 for q3w)
    nextCycleDate?: Date;
  };
  reminders: {
    enabled: boolean;
    advanceMinutes: number[];
    criticalReminder: boolean; // For chemo and critical meds
  };
  completionTracking: {
    trackCompletion: boolean;
    requireNotes: boolean;
    trackSideEffects: boolean;
    sideEffectOptions?: string[]; // Pre-defined side effects to track
  };
  treatmentContext: {
    createdBy: 'doctor' | 'patient' | 'nurse';
    createdById?: string;
    approvedBy?: string; // Doctor ID for approval
    isProtocolBased: boolean; // Following standard treatment protocol
    protocolName?: string;
  };
  createdAt: Date;
  updatedAt: Date;
  isActive: boolean;
}
```

### Diet Plan Model
```typescript
interface DietPlan {
  id: string;
  userId: string;
  name: string;
  description: string;
  createdBy: 'doctor' | 'nutritionist' | 'self';
  createdById?: string; // Doctor/nutritionist ID
  goals: {
    calories: number;
    protein: number;
    carbs: number;
    fat: number;
    fiber: number;
  };
  restrictions: string[]; // Allergies, dietary restrictions
  mealPlans: {
    breakfast: MealPlan;
    lunch: MealPlan;
    dinner: MealPlan;
    snacks: MealPlan[];
  };
  duration: {
    startDate: Date;
    endDate?: Date;
  };
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

### Medical Report Model
```typescript
interface MedicalReport {
  id: string;
  userId: string;
  uploadedBy: string; // User ID who uploaded
  reportType: 'blood_test' | 'imaging' | 'pathology' | 'consultation' | 'other';
  metadata: {
    testDate: Date;
    facility: string;
    doctorName?: string;
    reportTitle: string;
  };
  files: {
    originalFile: string; // S3 URL
    processedFile?: string; // OCR processed version
    thumbnails?: string[];
  };
  analysis: {
    hasAnalysis: boolean;
    analysisId?: string;
    doctorAuthorized: boolean;
    authorizedBy?: string; // Doctor ID
    authorizedAt?: Date;
  };
  privacy: {
    visibleToUser: boolean;
    visibleToDoctors: string[]; // Doctor IDs with access
  };
  tags: string[];
  createdAt: Date;
  updatedAt: Date;
}
```

### Report Analysis Model
```typescript
interface ReportAnalysis {
  id: string;
  reportId: string;
  userId: string;
  analysisType: 'ai_generated' | 'doctor_reviewed' | 'combined';
  content: {
    summary: string;
    keyFindings: string[];
    abnormalValues: {
      parameter: string;
      value: string;
      normalRange: string;
      significance: 'low' | 'medium' | 'high';
    }[];
    recommendations: string[];
    followUpRequired: boolean;
  };
  aiConfidence?: number; // 0-1 for AI analysis
  doctorReview?: {
    reviewedBy: string; // Doctor ID
    reviewedAt: Date;
    approved: boolean;
    notes: string;
  };
  accessControl: {
    patientCanView: boolean;
    authorizedBy: string; // Doctor ID
    authorizedAt: Date;
  };
  createdAt: Date;
  updatedAt: Date;
}
```

### Chemotherapy Cycle Model
```typescript
interface ChemotherapyCycle {
  id: string;
  patientId: string;
  routineId: string; // Links to the routine
  regimenName: string;
  cycleNumber: number;
  totalCycles: number;
  plannedDate: Date;
  actualDate?: Date;
  status: 'scheduled' | 'in_progress' | 'completed' | 'delayed' | 'cancelled';
  premedications: {
    drugName: string;
    completed: boolean;
    completedAt?: Date;
    notes?: string;
  }[];
  chemotherapyDrugs: {
    drugName: string;
    plannedDosage: string;
    actualDosage?: string;
    infusionStart?: Date;
    infusionEnd?: Date;
    completed: boolean;
    notes?: string;
  }[];
  sideEffects: {
    effect: string;
    severity: 'mild' | 'moderate' | 'severe';
    onset: Date;
    duration?: number; // Hours
    intervention?: string;
  }[];
  labResults: {
    testType: string;
    value: string;
    normalRange: string;
    testDate: Date;
    withinNormal: boolean;
  }[];
  nextCycleDate?: Date;
  delayReason?: string;
  doctorNotes: string;
  createdAt: Date;
  updatedAt: Date;
}
```

### Doctor Model
```typescript
interface Doctor {
  id: string;
  profile: {
    name: string;
    specialization: string[];
    licenseNumber: string;
    hospital: string;
    experience: number;
    contactInfo: {
      email: string;
      phone: string;
    };
  };
  patients: string[]; // Patient user IDs
  permissions: {
    canAnalyzeReports: boolean;
    canCreateDietPlans: boolean;
    canCreateRoutines: boolean;
    canViewProgress: boolean;
    canUpdateTreatmentPlan: boolean;
  };
  dashboardPreferences: {
    defaultView: 'overview' | 'progress' | 'alerts';
    alertThresholds: {
      missedMedications: number;
      abnormalValues: string[];
      routineAdherence: number;
    };
  };
  createdAt: Date;
  updatedAt: Date;
  isActive: boolean;
}
```

### Patient Progress Model
```typescript
interface PatientProgress {
  id: string;
  patientId: string;
  doctorId: string;
  period: {
    startDate: Date;
    endDate: Date;
  };
  metrics: {
    routineAdherence: {
      medication: number; // Percentage
      exercise: number;
      appointments: number;
      overall: number;
    };
    nutritionCompliance: {
      calorieGoals: number;
      dietPlanAdherence: number;
      restrictionCompliance: number;
    };
    reportTrends: {
      parameter: string;
      values: {
        date: Date;
        value: number;
        status: 'normal' | 'abnormal' | 'critical';
      }[];
      trend: 'improving' | 'stable' | 'declining';
    }[];
  };
  alerts: {
    id: string;
    type: 'missed_medication' | 'abnormal_value' | 'poor_adherence' | 'critical_result';
    severity: 'low' | 'medium' | 'high' | 'critical';
    message: string;
    triggered: Date;
    acknowledged: boolean;
    resolvedAt?: Date;
  }[];
  doctorNotes: {
    id: string;
    note: string;
    category: 'observation' | 'concern' | 'improvement' | 'instruction';
    createdAt: Date;
  }[];
  treatmentPlan: {
    currentPhase: string;
    nextMilestone: Date;
    goals: string[];
    modifications: {
      date: Date;
      change: string;
      reason: string;
    }[];
  };
  createdAt: Date;
  updatedAt: Date;
}
```

### Doctor Dashboard View Model
```typescript
interface DoctorDashboardView {
  doctorId: string;
  patients: {
    id: string;
    name: string;
    cancerType: string;
    journeyStage: string;
    lastActivity: Date;
    riskLevel: 'low' | 'medium' | 'high' | 'critical';
    recentAlerts: number;
    adherenceScore: number;
    nextAppointment?: Date;
    quickStats: {
      medicationAdherence: number;
      recentReports: number;
      missedAppointments: number;
    };
  }[];
  summary: {
    totalPatients: number;
    activeAlerts: number;
    criticalPatients: number;
    upcomingAppointments: number;
    recentReports: number;
  };
  recentActivity: {
    patientId: string;
    patientName: string;
    activity: string;
    timestamp: Date;
    requiresAttention: boolean;
  }[];
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, several properties can be consolidated to eliminate redundancy:

- **Encryption properties** (2.2, 4.2, 7.2) can be combined into a comprehensive data protection property
- **Privacy enforcement properties** (2.1, 2.4, 4.4) can be unified into a single privacy control property  
- **Language support properties** (1.2, 6.2, 6.4) can be consolidated into multilingual functionality
- **Offline functionality properties** (6.1, 6.5) can be combined into offline capability testing

### Core Properties

**Property 1: User Registration and Profile Management**
*For any* valid user registration data (email, phone, cancer type, journey stage), the system should create a user account and allow profile completion with medical information
**Validates: Requirements 1.1, 1.3, 1.4**

**Property 2: Doctor Access Control**
*For any* medical report or analysis, access should only be granted to patients themselves and doctors explicitly authorized by the treatment in-charge
**Validates: Requirements 2.1, 2.4**

**Property 3: Data Protection and Encryption**
*For any* sensitive medical data, the system should encrypt information at rest and in transit using industry-standard encryption, maintaining data integrity and confidentiality
**Validates: Requirements 2.2, 7.2**

**Property 4: Account Deletion Completeness**
*For any* user account deletion request, all associated medical data should be permanently removed from all system components within the specified timeframe
**Validates: Requirements 2.3**

**Property 5: Routine Planning and Chemotherapy Cycle Management**
*For any* routine creation including medications, chemotherapy cycles, and appointments, the system should properly schedule reminders, track cycle numbers, treatment phases, and completion according to oncology protocols
**Validates: Requirements 3.1, 3.2, 3.3, 3.4, 3.5**

**Property 6: Diet Plan Management**
*For any* diet plan created by doctors or nutritionists, the system should store nutritional goals, restrictions, and meal plans while allowing patient tracking of intake
**Validates: Requirements 3.3**

**Property 7: Medical Report Upload and Storage**
*For any* medical report upload, the system should securely store the file, extract metadata, and prepare it for potential AI analysis while maintaining patient privacy
**Validates: Requirements 5.1, 5.3**

**Property 8: AI Analysis Authorization**
*For any* medical report analysis request, the system should only provide AI-generated insights when explicitly authorized by the patient's treatment in-charge doctor
**Validates: Requirements 5.2, 5.4**

**Property 9: Multilingual Support and Translation**
*For any* user interaction in their preferred language from the supported set (22 Indian languages plus English), the system should provide proper language support including translation capabilities
**Validates: Requirements 6.2, 6.4**

**Property 10: Offline Functionality and Synchronization**
*For any* user action performed while offline (routine logging, meal tracking), the system should queue the action and synchronize it when connectivity is restored
**Validates: Requirements 6.1, 6.5**

**Property 11: Accessibility Feature Availability**
*For any* accessibility requirement (dark mode, large text, voice input), the system should provide functional accessibility options for users with different needs
**Validates: Requirements 6.3**

**Property 12: Secure Authentication and Audit Logging**
*For any* user authentication attempt and medical data access, the system should implement secure authentication and log appropriate events for audit purposes
**Validates: Requirements 7.1, 7.3**

**Property 13: System Scalability and Performance**
*For any* increase in user load, the system should automatically scale compute resources without service interruption while maintaining performance standards
**Validates: Requirements 8.1**

**Property 14: AI Service Integration**
*For any* AI-powered feature (report analysis, translation, risk assessment), the system should properly integrate with cloud-based ML services and provide accurate results
**Validates: Requirements 8.2**

**Property 16: Doctor Dashboard Patient Overview**
*For any* doctor accessing the dashboard, the system should provide accurate, real-time overview of all assigned patients including health status, risk levels, and recent activity
**Validates: Requirements 6.1, 6.2**

**Property 17: Treatment Progress Analytics**
*For any* patient progress request, the system should provide comprehensive analytics on routine adherence, nutrition compliance, and health trends with accurate calculations
**Validates: Requirements 6.3, 6.4**

**Property 18: Treatment Plan Management**
*For any* treatment plan modification by authorized doctors, the system should update patient routines, diet plans, and goals while maintaining treatment history
**Validates: Requirements 6.5**

## Error Handling

### Error Categories and Responses

#### 1. Authentication Errors
- **Invalid Credentials**: Return 401 with clear error message, implement rate limiting
- **Account Locked**: Return 423 with unlock instructions and support contact
- **MFA Failures**: Return 401 with MFA retry options and backup codes

#### 2. Authorization Errors  
- **Insufficient Permissions**: Return 403 with explanation of required permissions
- **Privacy Violations**: Block action and log security event for audit
- **Resource Access Denied**: Return 404 to prevent information disclosure

#### 3. Data Validation Errors
- **Invalid Input**: Return 400 with specific field validation errors
- **Missing Required Fields**: Return 422 with list of required fields
- **Data Format Errors**: Return 400 with format requirements and examples

#### 4. System Errors
- **Service Unavailable**: Return 503 with retry-after header and status page
- **Database Errors**: Return 500 with generic message, log detailed error internally
- **AI Service Failures**: Graceful degradation with fallback functionality

#### 5. Network and Connectivity Errors
- **Offline Mode**: Queue operations locally with user notification
- **Timeout Errors**: Retry with exponential backoff, inform user of delays
- **Sync Conflicts**: Present conflict resolution options to user

### Error Recovery Strategies

#### Graceful Degradation
- **AI Features Unavailable**: Disable semantic matching, use basic text search
- **Translation Service Down**: Show content in original language with notification
- **Image Upload Failures**: Allow text-only posts with retry option for media

#### Data Consistency
- **Partial Sync Failures**: Retry failed operations, maintain operation logs
- **Concurrent Modifications**: Implement optimistic locking with conflict resolution
- **Cache Invalidation**: Refresh stale data automatically with user notification

#### User Experience
- **Progress Indicators**: Show upload/sync progress for long operations
- **Error Messages**: Provide actionable error messages with next steps
- **Fallback Options**: Offer alternative actions when primary action fails

## Testing Strategy

### Dual Testing Approach

The testing strategy employs both unit testing and property-based testing to ensure comprehensive coverage:

- **Unit tests** verify specific examples, edge cases, and error conditions
- **Property tests** verify universal properties that should hold across all inputs
- Together they provide comprehensive coverage: unit tests catch concrete bugs, property tests verify general correctness

### Property-Based Testing Requirements

- **Testing Library**: Use `fast-check` for JavaScript/TypeScript property-based testing
- **Test Configuration**: Each property-based test must run a minimum of 100 iterations
- **Test Tagging**: Each property-based test must include a comment with the format: `**Feature: oncoo-community-platform, Property {number}: {property_text}**`
- **Property Implementation**: Each correctness property must be implemented by a single property-based test
- **Coverage**: Property tests should focus on universal behaviors that must hold across all valid inputs

### Unit Testing Requirements

Unit tests complement property tests by covering:
- **Specific Examples**: Concrete scenarios that demonstrate correct behavior
- **Edge Cases**: Boundary conditions and special cases
- **Integration Points**: Component interactions and API contracts
- **Error Conditions**: Specific error scenarios and recovery behaviors

### Test Categories

#### 1. Authentication and Security Tests
- **Unit Tests**: Login flows, MFA scenarios, session management
- **Property Tests**: Authentication token validity, encryption consistency
- **Security Tests**: Rate limiting, injection attacks, privilege escalation

#### 2. Privacy and Data Protection Tests
- **Unit Tests**: Privacy setting configurations, data access controls
- **Property Tests**: Row-level security enforcement, data encryption
- **Compliance Tests**: GDPR/DPDP compliance, audit trail verification

#### 3. Community and Content Tests
- **Unit Tests**: Post creation, community joining, moderation workflows
- **Property Tests**: Chronological ordering, content visibility rules
- **Integration Tests**: End-to-end user journeys, cross-component interactions

#### 4. AI and ML Feature Tests
- **Unit Tests**: Translation accuracy, semantic matching results
- **Property Tests**: AI service integration consistency, embedding generation
- **Performance Tests**: Response times, throughput under load

#### 5. Offline and Sync Tests
- **Unit Tests**: Offline queue management, conflict resolution
- **Property Tests**: Data consistency after sync, offline functionality preservation
- **Network Tests**: Various connectivity scenarios, sync reliability

#### 6. Accessibility and Internationalization Tests
- **Unit Tests**: Language switching, accessibility feature activation
- **Property Tests**: Multi-language support consistency, voice processing accuracy
- **Usability Tests**: Screen reader compatibility, keyboard navigation

### Continuous Testing Pipeline

#### Pre-commit Hooks
- Run unit tests for modified components
- Execute linting and code quality checks
- Validate API contract compliance

#### CI/CD Pipeline
- **Build Stage**: Compile TypeScript, bundle assets, run static analysis
- **Test Stage**: Execute full unit test suite, run property-based tests
- **Security Stage**: Dependency scanning, SAST analysis, secret detection
- **Integration Stage**: API testing, database migration validation
- **Performance Stage**: Load testing, memory leak detection

#### Production Monitoring
- **Health Checks**: Endpoint availability, database connectivity
- **Performance Metrics**: Response times, error rates, throughput
- **Security Monitoring**: Failed authentication attempts, suspicious activity
- **User Experience**: Real user monitoring, error tracking, performance insights