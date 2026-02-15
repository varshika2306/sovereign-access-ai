# Design Document: Sovereign Access AI

## Overview

Sovereign Access AI is a multilingual, AI-powered platform that connects underserved Indian communities with government schemes and public programs. The system uses natural language processing, eligibility matching algorithms, and a conversational interface to help users discover, understand, and apply for benefits they qualify for.

The platform is designed for low-bandwidth environments and users with varying levels of digital literacy. It provides personalized scheme recommendations, step-by-step application guidance, document checklists, and application tracking—all accessible through text or voice in 10 Indian languages.

Key architectural principles:
- **Accessibility-first**: Support for voice, low-bandwidth, and screen readers
- **Multilingual by design**: All components support 10 Indian languages
- **Offline-capable**: Critical information cached for offline access
- **Privacy-focused**: End-to-end encryption and DPDPA compliance
- **Scalable**: Designed to serve millions of users across India

## Architecture

The system follows a microservices architecture with the following high-level components:

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Layer                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Web App      │  │ Mobile App   │  │ Voice IVR    │      │
│  │ (PWA)        │  │ (Android)    │  │ Interface    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     API Gateway                              │
│              (Authentication, Rate Limiting)                 │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Conversation │   │  Eligibility │   │    User      │
│   Service    │   │    Engine    │   │  Management  │
└──────────────┘   └──────────────┘   └──────────────┘
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   NLP/LLM    │   │   Scheme     │   │    Auth      │
│   Service    │   │   Database   │   │   Service    │
└──────────────┘   └──────────────┘   └──────────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Supporting Services                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Translation  │  │ Notification │  │   Cache      │      │
│  │   Service    │  │   Service    │  │   Layer      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

**Client Layer:**
- Progressive Web App (PWA) for cross-platform web access
- Native Android app for mobile users
- Voice IVR interface for phone-based access
- Offline caching and sync capabilities

**API Gateway:**
- Request routing and load balancing
- Authentication token validation
- Rate limiting and DDoS protection
- Request/response logging

**Conversation Service:**
- Natural language understanding and intent classification
- Context management across conversation turns
- Response generation in user's language
- Integration with NLP/LLM service

**Eligibility Engine:**
- Rule-based matching of user profiles to scheme criteria
- Relevance scoring and ranking
- Missing information identification
- Real-time eligibility evaluation

**User Management Service:**
- Profile CRUD operations
- Data encryption and privacy controls
- Consent management
- Data export and deletion

**Scheme Database:**
- Centralized repository of government schemes
- Eligibility criteria, benefits, and application processes
- Regional variations and updates
- Search and filtering capabilities

**NLP/LLM Service:**
- Multilingual text understanding (10 languages)
- Speech-to-text and text-to-speech
- Intent extraction and entity recognition
- Conversational response generation

**Translation Service:**
- Real-time translation between supported languages
- Preservation of technical terms and scheme names
- Caching of common translations

**Notification Service:**
- Multi-channel delivery (SMS, email, in-app)
- Scheduled reminders and deadline alerts
- User preference management
- Delivery status tracking

**Cache Layer:**
- Redis for session data and frequently accessed schemes
- CDN for static assets
- Local storage for offline capability

## Components and Interfaces

### 1. Conversation Service

**Purpose:** Manages conversational interactions with users, understanding intents and generating contextual responses.

**Key Interfaces:**

```typescript
interface ConversationService {
  // Process user message and return response
  processMessage(
    userId: string,
    message: string,
    language: Language,
    sessionId: string
  ): Promise<ConversationResponse>;
  
  // Get conversation history
  getHistory(sessionId: string): Promise<Message[]>;
  
  // Clear conversation context
  resetContext(sessionId: string): Promise<void>;
}

interface ConversationResponse {
  text: string;
  intent: Intent;
  entities: Entity[];
  suggestedActions: Action[];
  requiresClarification: boolean;
}

enum Intent {
  DISCOVER_SCHEMES,
  CHECK_ELIGIBILITY,
  GET_APPLICATION_GUIDE,
  TRACK_APPLICATION,
  UPDATE_PROFILE,
  COMPARE_SCHEMES,
  GET_HELP
}
```

**Dependencies:**
- NLP/LLM Service for language understanding
- Eligibility Engine for scheme matching
- User Management for profile access
- Translation Service for multilingual support

### 2. Eligibility Engine

**Purpose:** Matches users to schemes based on eligibility criteria and ranks results by relevance.

**Key Interfaces:**

```typescript
interface EligibilityEngine {
  // Find schemes matching user profile
  findEligibleSchemes(
    userProfile: UserProfile,
    filters?: SchemeFilters
  ): Promise<RankedScheme[]>;
  
  // Check eligibility for specific scheme
  checkEligibility(
    userProfile: UserProfile,
    schemeId: string
  ): Promise<EligibilityResult>;
  
  // Identify missing information for better matching
  getMissingInformation(
    userProfile: UserProfile
  ): Promise<ProfileGap[]>;
}

interface RankedScheme {
  scheme: Scheme;
  relevanceScore: number;
  matchedCriteria: string[];
  missingCriteria: string[];
  applicationDeadline?: Date;
}

interface EligibilityResult {
  eligible: boolean;
  matchedCriteria: Criterion[];
  failedCriteria: Criterion[];
  confidence: number;
}
```

**Matching Algorithm:**
1. Extract user attributes from profile (location, category, income, etc.)
2. For each scheme, evaluate eligibility criteria as boolean expressions
3. Calculate relevance score based on:
   - Number of matched criteria (weight: 40%)
   - User category alignment (weight: 30%)
   - Application deadline proximity (weight: 20%)
   - Scheme popularity in user's region (weight: 10%)
4. Rank schemes by relevance score (descending)
5. Return top N schemes with match details

### 3. User Management Service

**Purpose:** Manages user profiles, authentication, and data privacy.

**Key Interfaces:**

```typescript
interface UserManagementService {
  // Create new user profile
  createProfile(
    phoneNumber: string,
    otp: string,
    profileData: ProfileData
  ): Promise<User>;
  
  // Update user profile
  updateProfile(
    userId: string,
    updates: Partial<ProfileData>
  ): Promise<User>;
  
  // Get user profile
  getProfile(userId: string): Promise<User>;
  
  // Delete user account and data
  deleteAccount(userId: string): Promise<void>;
  
  // Export user data
  exportData(userId: string): Promise<UserDataExport>;
}

interface User {
  id: string;
  phoneNumber: string;
  profile: ProfileData;
  preferences: UserPreferences;
  createdAt: Date;
  updatedAt: Date;
}

interface ProfileData {
  name: string;
  location: Location;
  category: UserCategory;
  language: Language;
  demographics?: Demographics;
  businessInfo?: BusinessInfo;
}

enum UserCategory {
  GRASSROOTS_INNOVATOR,
  SHG_MEMBER,
  STUDENT_ENTREPRENEUR,
  RURAL_MSME,
  FIRST_GEN_DIGITAL_USER
}
```

**Security Measures:**
- AES-256 encryption for sensitive profile data
- Phone number hashing for lookup
- JWT tokens with 24-hour expiration
- Rate limiting on authentication endpoints
- Audit logging for data access

### 4. Scheme Database Service

**Purpose:** Stores and retrieves government scheme information with search and filtering capabilities.

**Key Interfaces:**

```typescript
interface SchemeDatabase {
  // Add new scheme
  addScheme(scheme: SchemeData): Promise<Scheme>;
  
  // Update existing scheme
  updateScheme(schemeId: string, updates: Partial<SchemeData>): Promise<Scheme>;
  
  // Get scheme by ID
  getScheme(schemeId: string, language: Language): Promise<Scheme>;
  
  // Search schemes
  searchSchemes(query: string, filters: SchemeFilters): Promise<Scheme[]>;
  
  // Get schemes by region
  getSchemesByRegion(state: string, district?: string): Promise<Scheme[]>;
}

interface Scheme {
  id: string;
  name: Record<Language, string>;
  description: Record<Language, string>;
  eligibilityCriteria: Criterion[];
  benefits: string[];
  applicationProcess: ApplicationStep[];
  requiredDocuments: Document[];
  deadline?: Date;
  officialLink: string;
  region: Region;
  category: SchemeCategory[];
  status: SchemeStatus;
}

interface Criterion {
  field: string;
  operator: ComparisonOperator;
  value: any;
  description: Record<Language, string>;
}

enum ComparisonOperator {
  EQUALS,
  NOT_EQUALS,
  GREATER_THAN,
  LESS_THAN,
  IN_RANGE,
  CONTAINS,
  ONE_OF
}
```

**Data Model:**
- Schemes stored in PostgreSQL for structured queries
- Multilingual content stored as JSONB
- Full-text search using PostgreSQL's tsvector
- Indexed on region, category, and deadline for fast filtering

### 5. NLP/LLM Service

**Purpose:** Provides natural language understanding, speech processing, and conversational AI capabilities.

**Key Interfaces:**

```typescript
interface NLPService {
  // Extract intent and entities from text
  analyzeText(
    text: string,
    language: Language,
    context?: ConversationContext
  ): Promise<NLPResult>;
  
  // Generate conversational response
  generateResponse(
    intent: Intent,
    entities: Entity[],
    context: ConversationContext,
    language: Language
  ): Promise<string>;
  
  // Convert speech to text
  speechToText(
    audioData: Buffer,
    language: Language
  ): Promise<string>;
  
  // Convert text to speech
  textToSpeech(
    text: string,
    language: Language
  ): Promise<Buffer>;
}

interface NLPResult {
  intent: Intent;
  entities: Entity[];
  confidence: number;
  sentiment?: Sentiment;
}

interface Entity {
  type: EntityType;
  value: string;
  confidence: number;
}

enum EntityType {
  LOCATION,
  SCHEME_NAME,
  USER_CATEGORY,
  DATE,
  AMOUNT,
  DOCUMENT_TYPE
}
```

**Implementation Approach:**
- Use pre-trained multilingual models (mBERT or IndicBERT for Indian languages)
- Fine-tune on government scheme domain data
- Intent classification using transformer-based classifier
- Entity extraction using NER models
- Speech processing using Whisper or IndicWav2Vec
- Response generation using template-based approach with LLM fallback

### 6. Application Guide Service

**Purpose:** Provides step-by-step guidance for scheme applications.

**Key Interfaces:**

```typescript
interface ApplicationGuideService {
  // Get application guide for scheme
  getGuide(
    schemeId: string,
    userProfile: UserProfile,
    language: Language
  ): Promise<ApplicationGuide>;
  
  // Mark step as complete
  completeStep(
    userId: string,
    schemeId: string,
    stepId: string
  ): Promise<void>;
  
  // Get user's application progress
  getProgress(
    userId: string,
    schemeId: string
  ): Promise<ApplicationProgress>;
}

interface ApplicationGuide {
  schemeId: string;
  steps: ApplicationStep[];
  estimatedTime: number;
  documentChecklist: DocumentChecklist;
  tips: string[];
}

interface ApplicationStep {
  id: string;
  title: string;
  description: string;
  estimatedMinutes: number;
  portalLink?: string;
  formFields?: FormField[];
  examples?: string[];
}

interface DocumentChecklist {
  schemeId: string;
  documents: RequiredDocument[];
  personalizedNotes: string[];
}

interface RequiredDocument {
  name: string;
  format: string[];
  issuingAuthority: string;
  validityRequirement?: string;
  isMandatory: boolean;
  acquisitionGuide?: string;
}
```

### 7. Notification Service

**Purpose:** Sends multi-channel notifications to users about schemes, deadlines, and updates.

**Key Interfaces:**

```typescript
interface NotificationService {
  // Send notification
  sendNotification(
    userId: string,
    notification: Notification
  ): Promise<DeliveryStatus>;
  
  // Schedule notification
  scheduleNotification(
    userId: string,
    notification: Notification,
    scheduledTime: Date
  ): Promise<string>;
  
  // Update user notification preferences
  updatePreferences(
    userId: string,
    preferences: NotificationPreferences
  ): Promise<void>;
}

interface Notification {
  type: NotificationType;
  title: string;
  body: string;
  actionUrl?: string;
  priority: Priority;
}

enum NotificationType {
  NEW_SCHEME_MATCH,
  DEADLINE_REMINDER,
  APPLICATION_STATUS_UPDATE,
  SCHEME_UPDATE,
  SYSTEM_ANNOUNCEMENT
}

interface NotificationPreferences {
  channels: Channel[];
  frequency: Frequency;
  quietHours?: TimeRange;
}

enum Channel {
  SMS,
  EMAIL,
  IN_APP,
  WHATSAPP
}
```

**Delivery Strategy:**
- SMS for critical notifications (deadlines, status updates)
- In-app for general updates
- Email for detailed information
- WhatsApp for rich media (where available)
- Retry logic: 3 attempts with exponential backoff
- Delivery tracking and analytics

### 8. Partner Dashboard Service

**Purpose:** Provides analytics and user support tools for NGOs and government officials.

**Key Interfaces:**

```typescript
interface PartnerDashboardService {
  // Get aggregate analytics
  getAnalytics(
    partnerId: string,
    filters: AnalyticsFilters
  ): Promise<Analytics>;
  
  // Search users (with consent)
  searchUsers(
    partnerId: string,
    query: string
  ): Promise<UserSummary[]>;
  
  // Get user's scheme matches (with consent)
  getUserSchemes(
    partnerId: string,
    userId: string
  ): Promise<RankedScheme[]>;
  
  // Generate report
  generateReport(
    partnerId: string,
    reportType: ReportType,
    parameters: ReportParameters
  ): Promise<Report>;
}

interface Analytics {
  totalUsers: number;
  newUsersThisMonth: number;
  schemesDiscovered: number;
  applicationsStarted: number;
  successStories: number;
  topSchemes: SchemeStats[];
  regionalBreakdown: RegionStats[];
  categoryBreakdown: CategoryStats[];
}
```

## Data Models

### User Profile

```typescript
interface UserProfile {
  userId: string;
  phoneNumber: string; // hashed
  name: string; // encrypted
  location: {
    state: string;
    district: string;
    pincode?: string;
    coordinates?: {
      latitude: number;
      longitude: number;
    };
  };
  category: UserCategory;
  language: Language;
  demographics?: {
    age?: number;
    gender?: Gender;
    caste?: CasteCategory;
    income?: IncomeRange;
    education?: EducationLevel;
  };
  businessInfo?: {
    type: BusinessType;
    sector: string;
    employeeCount?: number;
    annualRevenue?: number;
    registrationStatus: RegistrationStatus;
  };
  preferences: {
    notificationChannels: Channel[];
    notificationFrequency: Frequency;
    lowBandwidthMode: boolean;
    voicePreferred: boolean;
  };
  consent: {
    dataCollection: boolean;
    partnerAccess: boolean;
    communicationOptIn: boolean;
    consentDate: Date;
  };
  metadata: {
    createdAt: Date;
    updatedAt: Date;
    lastLoginAt: Date;
    profileCompleteness: number; // 0-100
  };
}
```

### Scheme

```typescript
interface Scheme {
  schemeId: string;
  name: Record<Language, string>;
  shortDescription: Record<Language, string>;
  fullDescription: Record<Language, string>;
  
  eligibility: {
    criteria: Criterion[];
    targetCategories: UserCategory[];
    ageRange?: { min: number; max: number };
    incomeLimit?: number;
    locationRestrictions?: {
      states?: string[];
      districts?: string[];
      ruralOnly?: boolean;
    };
  };
  
  benefits: {
    type: BenefitType[];
    monetaryAmount?: number;
    description: Record<Language, string>;
  };
  
  application: {
    process: ApplicationStep[];
    requiredDocuments: RequiredDocument[];
    portalUrl: string;
    helplineNumber?: string;
    estimatedProcessingDays: number;
  };
  
  timeline: {
    startDate?: Date;
    endDate?: Date;
    applicationDeadline?: Date;
    isOngoing: boolean;
  };
  
  metadata: {
    ministry: string;
    department: string;
    officialNotificationUrl: string;
    lastUpdated: Date;
    popularity: number; // based on applications
    successRate?: number;
  };
}
```

### Conversation Session

```typescript
interface ConversationSession {
  sessionId: string;
  userId: string;
  language: Language;
  messages: Message[];
  context: {
    currentIntent?: Intent;
    activeSchemeId?: string;
    collectedEntities: Record<string, any>;
    clarificationNeeded?: string[];
  };
  metadata: {
    startedAt: Date;
    lastActivityAt: Date;
    messageCount: number;
  };
}

interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  intent?: Intent;
  entities?: Entity[];
}
```

### Application Tracking

```typescript
interface ApplicationTracking {
  trackingId: string;
  userId: string;
  schemeId: string;
  applicationReferenceNumber?: string;
  status: ApplicationStatus;
  timeline: ApplicationEvent[];
  documents: {
    documentType: string;
    uploaded: boolean;
    uploadedAt?: Date;
  }[];
  notes: string[];
  createdAt: Date;
  updatedAt: Date;
}

enum ApplicationStatus {
  NOT_STARTED,
  IN_PROGRESS,
  SUBMITTED,
  UNDER_REVIEW,
  APPROVED,
  REJECTED,
  WITHDRAWN
}

interface ApplicationEvent {
  status: ApplicationStatus;
  timestamp: Date;
  description: string;
  source: 'user' | 'system' | 'portal';
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

