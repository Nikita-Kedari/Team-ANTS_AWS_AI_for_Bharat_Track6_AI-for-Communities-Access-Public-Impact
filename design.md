# Design Document: AI Community Access Assistant

## Overview

The AI Community Access Assistant is a multi-modal civic access platform that bridges the gap between public systems and citizens under real constraints. The system employs a layered architecture with context-aware AI, adaptive interfaces, and offline-first design principles to serve users across diverse literacy levels, connectivity constraints, and accessibility needs.

### Key Design Principles

1. **Progressive Enhancement**: Core functionality works on feature phones with 2G; enhanced features activate on better devices/networks
2. **Context-First Architecture**: User context drives all personalization, from UI complexity to content explanation style
3. **Offline-First Data Strategy**: Essential data cached locally; sync happens opportunistically
4. **Multi-Modal by Default**: Every feature accessible via text, voice, and visual interfaces
5. **Trust Through Transparency**: All data sources, confidence scores, and update timestamps visible to users
6. **Privacy by Design**: Minimal data collection; anonymous mode as first-class citizen

### Technology Stack Considerations

- **Frontend**: Progressive Web App (PWA) with adaptive rendering based on device capabilities
- **Backend**: Microservices architecture for independent scaling of AI, data, and interface services
- **AI/ML**: Hybrid approach combining rule-based eligibility logic with LLM for natural language understanding and explanation generation
- **Data Storage**: Multi-tier caching (device, edge, cloud) with eventual consistency model
- **Voice Processing**: Speech-to-text and text-to-speech with regional language support
- **Messaging Integration**: WhatsApp Business API, SMS gateway, IVR system integration

## Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   PWA    │  │ WhatsApp │  │   SMS    │  │   IVR    │       │
│  │  (Web/   │  │ Interface│  │ Interface│  │  System  │       │
│  │ Mobile)  │  │          │  │          │  │          │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┼──────────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                      │
        ┌─────────────▼─────────────────────────────────┐
        │         API Gateway Layer                      │
        │  - Request routing                             │
        │  - Authentication/Authorization                │
        │  - Rate limiting                               │
        │  - Protocol translation                        │
        └─────────────┬─────────────────────────────────┘
                      │
        ┌─────────────▼─────────────────────────────────┐
        │         Service Layer                          │
        │  ┌──────────────┐  ┌──────────────┐          │
        │  │   Context    │  │  Eligibility │          │
        │  │   Service    │  │   Service    │          │
        │  └──────────────┘  └──────────────┘          │
        │  ┌──────────────┐  ┌──────────────┐          │
        │  │     AI       │  │    Content   │          │
        │  │   Service    │  │   Service    │          │
        │  └──────────────┘  └──────────────┘          │
        │  ┌──────────────┐  ┌──────────────┐          │
        │  │ Notification │  │   Analytics  │          │
        │  │   Service    │  │   Service    │          │
        │  └──────────────┘  └──────────────┘          │
        └─────────────┬─────────────────────────────────┘
                      │
        ┌─────────────▼─────────────────────────────────┐
        │         Data Layer                             │
        │  ┌──────────────┐  ┌──────────────┐          │
        │  │   Scheme     │  │     User     │          │
        │  │  Database    │  │   Database   │          │
        │  └──────────────┘  └──────────────┘          │
        │  ┌──────────────┐  ┌──────────────┐          │
        │  │   Content    │  │   Analytics  │          │
        │  │    Cache     │  │  Data Store  │          │
        │  └──────────────┘  └──────────────┘          │
        └───────────────────────────────────────────────┘
```

### Data Flow Architecture


**User Query Flow:**
1. User submits query via any interface (PWA, WhatsApp, SMS, IVR)
2. API Gateway authenticates and routes to appropriate service
3. Context Service enriches query with user profile and location data
4. AI Service processes query using NLU to determine intent
5. Eligibility Service calculates matches based on context and rules
6. Content Service retrieves and adapts explanations for user's literacy/language
7. Response formatted for user's interface and returned
8. Notification Service schedules follow-up reminders if applicable

**Offline-First Flow:**
1. Client maintains local cache of user profile, saved schemes, and recent searches
2. Queries first check local cache; if hit, return immediately with "offline" indicator
3. Background sync service queues actions when offline
4. When connectivity restored, sync service uploads queued actions and downloads updates
5. Conflict resolution prioritizes most recent user action

### Scalability Considerations

- **Horizontal Scaling**: All services stateless and containerized for independent scaling
- **Caching Strategy**: Multi-tier caching (CDN, Redis, local storage) reduces database load
- **Database Sharding**: User data sharded by region; scheme data replicated across regions
- **Async Processing**: Heavy operations (AI inference, analytics) processed asynchronously via message queue
- **Edge Computing**: Static content and frequently accessed schemes served from edge locations

## Components and Interfaces

### 1. Context Service

**Responsibility**: Manages user context profiles and provides context-aware enrichment for all requests.

**Key Functions:**
- `createProfile(userData)`: Creates new user context profile
- `updateProfile(userId, updates)`: Updates existing profile
- `getContext(userId)`: Retrieves complete context for a user
- `inferMissingData(partialProfile)`: Makes reasonable inferences for incomplete profiles
- `anonymizeContext(context)`: Strips identifying information for anonymous mode

**Data Model:**
```typescript
interface UserContext {
  userId: string | null;  // null for anonymous users
  location: {
    state: string;
    district: string;
    pincode: string;
    coordinates?: { lat: number; lon: number };
  };
  demographics: {
    age?: number;
    gender?: string;
    occupation?: string;
    education?: string;
    income?: string;
  };
  accessibility: {
    literacyLevel: 'low' | 'medium' | 'high';
    disabilities?: string[];
    preferredLanguage: string;
    preferredInterface: 'text' | 'voice' | 'mixed';
  };
  device: {
    type: 'feature_phone' | 'smartphone' | 'tablet' | 'desktop';
    connectivity: '2g' | '3g' | '4g' | 'wifi';
    capabilities: string[];
  };
  preferences: {
    notificationChannels: string[];
    reminderFrequency: string;
    privacyMode: 'full' | 'anonymous';
  };
  createdAt: Date;
  lastUpdated: Date;
}
```

**Interface Contracts:**
- Input: User-provided data (may be incomplete)
- Output: Enriched context object with inferred values
- Error Handling: Returns partial context if enrichment fails; never blocks user flow

### 2. Eligibility Service

**Responsibility**: Calculates eligibility scores and matches users to schemes based on rules and context.

**Key Functions:**
- `calculateEligibility(context, schemeId)`: Returns eligibility score (0-100) and detailed breakdown
- `findMatchingSchemes(context, filters)`: Returns ranked list of schemes matching user context
- `explainEligibility(context, schemeId)`: Generates human-readable explanation of eligibility status
- `suggestImprovements(context, schemeId)`: Recommends changes to become eligible

**Eligibility Calculation Algorithm:**
```
1. Load scheme requirements (age, income, location, occupation, etc.)
2. For each requirement:
   a. Check if user context satisfies requirement
   b. Assign weight based on requirement criticality
   c. Calculate partial score if requirement partially met
3. Aggregate weighted scores to produce final eligibility percentage
4. Generate breakdown showing which requirements met/not met
5. If score < 100%, identify minimum changes needed to reach 100%
```

**Data Model:**
```typescript
interface Scheme {
  id: string;
  name: string;
  description: string;
  domain: string[];  // education, welfare, employment, etc.
  requirements: Requirement[];
  benefits: string;
  applicationProcess: Step[];
  documents: Document[];
  deadlines?: Date[];
  contactInfo: ContactInfo;
  geography: {
    level: 'national' | 'state' | 'district';
    applicableRegions: string[];
  };
  source: {
    url: string;
    lastVerified: Date;
    confidence: number;
  };
}

interface Requirement {
  field: string;  // age, income, occupation, etc.
  operator: 'equals' | 'range' | 'in' | 'not_in';
  value: any;
  weight: number;  // 0-1, importance of this requirement
  mandatory: boolean;
}

interface EligibilityResult {
  schemeId: string;
  score: number;  // 0-100
  breakdown: {
    requirement: string;
    met: boolean;
    userValue: any;
    requiredValue: any;
  }[];
  recommendations: string[];  // How to become eligible
}
```

**Interface Contracts:**
- Input: UserContext + SchemeId or Filters
- Output: EligibilityResult or ranked list of schemes
- Performance: < 100ms for single scheme, < 500ms for full search

### 3. AI Service

**Responsibility**: Natural language understanding, intent detection, and culturally-adapted explanation generation.

**Key Functions:**
- `detectIntent(query, context)`: Determines user intent from natural language query
- `generateExplanation(scheme, context)`: Creates culturally-adapted explanation
- `answerQuestion(question, context)`: Provides conversational answers
- `simplifyText(text, literacyLevel)`: Adapts text complexity to user's literacy
- `translateWithContext(text, targetLanguage, context)`: Culturally-aware translation

**AI Architecture:**
```
┌─────────────────────────────────────────────────────┐
│              AI Service Components                   │
│                                                      │
│  ┌──────────────────┐    ┌──────────────────┐     │
│  │  Intent Detector │    │  NLU Processor   │     │
│  │  (Rule + ML)     │    │  (LLM-based)     │     │
│  └──────────────────┘    └──────────────────┘     │
│                                                      │
│  ┌──────────────────┐    ┌──────────────────┐     │
│  │   Explanation    │    │   Translation    │     │
│  │   Generator      │    │   Engine         │     │
│  │   (LLM-based)    │    │  (Hybrid)        │     │
│  └──────────────────┘    └──────────────────┘     │
│                                                      │
│  ┌──────────────────────────────────────────┐     │
│  │      Cultural Adaptation Layer            │     │
│  │  - Regional analogies database            │     │
│  │  - Occupation-specific examples           │     │
│  │  - Dialect handling                       │     │
│  └──────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────┘
```

**Explanation Generation Strategy:**
1. Retrieve base scheme description
2. Analyze user context (occupation, education, location)
3. Select appropriate cultural analogies from database
4. Adjust language complexity based on literacy level
5. Generate examples relevant to user's situation
6. Format for user's preferred interface (text/voice)

**Example Adaptation:**
- **For Farmer**: "This scheme provides ₹6000 per year, paid in 3 installments of ₹2000 each - like getting money after each harvest season"
- **For Student**: "This scholarship covers your college fees and gives ₹1000 per month for books and living expenses"
- **For Daily Wage Worker**: "You'll get ₹500 every week directly in your bank account - no need to visit any office"

**Interface Contracts:**
- Input: Query/Content + UserContext
- Output: Intent classification or adapted content
- Latency: < 2s for explanation generation, < 500ms for intent detection

### 4. Content Service

**Responsibility**: Manages scheme data, content delivery, and trust/verification metadata.

**Key Functions:**
- `getScheme(schemeId, context)`: Retrieves scheme with context-adapted content
- `searchSchemes(query, context, filters)`: Full-text search with personalization
- `getActionGuide(schemeId, context)`: Generates step-by-step application guide
- `getDocumentChecklist(schemeId, context)`: Creates personalized document list
- `verifyFreshness(schemeId)`: Checks if scheme data is current

**Content Adaptation Pipeline:**
```
Raw Scheme Data → Context Analysis → Language Selection → 
Complexity Adjustment → Cultural Adaptation → Format for Interface → 
Add Trust Badges → Return to User
```

**Trust Badge Generation:**
```typescript
interface TrustBadge {
  source: string;  // Official government website URL
  lastUpdated: Date;
  confidence: 'high' | 'medium' | 'low';
  verificationStatus: 'verified' | 'pending' | 'outdated';
  scamWarning?: string;
}

function generateTrustBadge(scheme: Scheme): TrustBadge {
  const daysSinceUpdate = (Date.now() - scheme.source.lastVerified) / (1000 * 60 * 60 * 24);
  
  let confidence: 'high' | 'medium' | 'low';
  let verificationStatus: 'verified' | 'pending' | 'outdated';
  
  if (daysSinceUpdate < 30) {
    confidence = 'high';
    verificationStatus = 'verified';
  } else if (daysSinceUpdate < 90) {
    confidence = 'medium';
    verificationStatus = 'verified';
  } else {
    confidence = 'low';
    verificationStatus = 'outdated';
  }
  
  return {
    source: scheme.source.url,
    lastUpdated: scheme.source.lastVerified,
    confidence,
    verificationStatus,
    scamWarning: getScamWarning(scheme.id)
  };
}
```

**Interface Contracts:**
- Input: SchemeId/Query + UserContext
- Output: Adapted content with trust metadata
- Caching: Aggressive caching with context-aware cache keys

### 5. Notification Service

**Responsibility**: Manages reminders, alerts, and multi-channel notification delivery.

**Key Functions:**
- `scheduleReminder(userId, milestone, deadline, preferences)`: Creates reminder schedule
- `sendNotification(userId, message, channel)`: Sends immediate notification
- `updatePreferences(userId, preferences)`: Updates notification settings
- `getScheduledReminders(userId)`: Retrieves upcoming reminders

**Notification Scheduling Strategy:**
```
For each milestone with deadline:
1. Calculate reminder dates: deadline - 7 days, - 3 days, - 1 day
2. Check user's notification preferences (channels, quiet hours)
3. Create scheduled jobs in notification queue
4. When reminder time arrives:
   a. Check if milestone already completed
   b. If not, send via preferred channels
   c. Log delivery status
   d. Retry failed deliveries with exponential backoff
```

**Multi-Channel Delivery:**
```typescript
interface NotificationChannel {
  type: 'push' | 'sms' | 'whatsapp' | 'ivr';
  priority: number;
  costPerMessage: number;
  deliveryRate: number;
}

function selectChannel(preferences: UserPreferences, urgency: 'high' | 'medium' | 'low'): NotificationChannel {
  // Filter by user preferences
  const availableChannels = ALL_CHANNELS.filter(c => preferences.channels.includes(c.type));
  
  // For high urgency, use highest priority channel
  if (urgency === 'high') {
    return availableChannels.sort((a, b) => b.priority - a.priority)[0];
  }
  
  // For medium/low urgency, optimize for cost and delivery rate
  return availableChannels.sort((a, b) => 
    (b.deliveryRate / b.costPerMessage) - (a.deliveryRate / a.costPerMessage)
  )[0];
}
```

**Interface Contracts:**
- Input: UserId + Message + Preferences
- Output: Delivery confirmation or scheduled job ID
- Reliability: At-least-once delivery guarantee with deduplication

### 6. Analytics Service

**Responsibility**: Collects usage data, generates insights, and provides admin dashboard data.

**Key Functions:**
- `trackEvent(event, context)`: Records user interaction
- `generateReport(reportType, filters)`: Creates analytics report
- `identifyDemandGaps(region, domain)`: Finds unmet needs
- `getUsageTrends(timeRange, dimensions)`: Returns aggregated usage patterns

**Privacy-Preserving Analytics:**
```
1. All events anonymized before storage
2. Personal identifiers hashed with salt
3. Aggregation happens at collection time
4. Individual user data never exposed in reports
5. Minimum aggregation threshold: 10 users
```

**Key Metrics Tracked:**
- Search queries and result relevance
- Scheme views and application starts
- Completion rates by scheme and user segment
- Channel usage (web, WhatsApp, SMS, IVR)
- Geographic distribution of demand
- Common drop-off points in application flows

**Interface Contracts:**
- Input: Event data or report parameters
- Output: Anonymized aggregated insights
- Privacy: Zero PII in analytics database

## Data Models

### Core Entities

**User Profile:**
```typescript
interface UserProfile {
  id: string;
  context: UserContext;  // From Context Service
  savedSchemes: string[];  // Scheme IDs
  applications: Application[];
  preferences: UserPreferences;
  createdAt: Date;
  lastActive: Date;
}

interface Application {
  schemeId: string;
  status: 'saved' | 'started' | 'in_progress' | 'submitted' | 'completed';
  milestones: Milestone[];
  startedAt: Date;
  lastUpdated: Date;
}

interface Milestone {
  id: string;
  title: string;
  description: string;
  deadline?: Date;
  completed: boolean;
  completedAt?: Date;
  order: number;
}
```

**Scheme (Extended):**
```typescript
interface Scheme {
  // Core fields from Eligibility Service
  id: string;
  name: string;
  description: string;
  domain: string[];
  requirements: Requirement[];
  benefits: string;
  
  // Application process
  applicationProcess: Step[];
  documents: Document[];
  estimatedTime: string;  // "2-3 weeks"
  
  // Geographic and temporal
  geography: Geography;
  deadlines: Deadline[];
  
  // Contact and resources
  contactInfo: ContactInfo;
  resources: Resource[];
  
  // Trust and verification
  source: Source;
  scamWarnings: ScamWarning[];
  
  // Metadata
  createdAt: Date;
  updatedAt: Date;
  viewCount: number;
  applicationCount: number;
}

interface Step {
  order: number;
  title: string;
  description: string;
  estimatedDuration: string;
  type: 'online' | 'offline' | 'hybrid';
  url?: string;
  location?: string;
}

interface Document {
  name: string;
  description: string;
  required: boolean;
  alternatives: string[];
  howToObtain: string;
  estimatedCost: number;
  estimatedTime: string;
}

interface ContactInfo {
  phone: string[];
  email: string[];
  website: string;
  officeAddress: string[];
  helplineHours: string;
}

interface Resource {
  type: 'office' | 'training_center' | 'help_desk';
  name: string;
  address: string;
  coordinates: { lat: number; lon: number };
  phone: string;
  hours: string;
}
```

**Notification:**
```typescript
interface Notification {
  id: string;
  userId: string;
  type: 'reminder' | 'alert' | 'update' | 'new_scheme';
  title: string;
  message: string;
  channel: 'push' | 'sms' | 'whatsapp' | 'ivr';
  priority: 'high' | 'medium' | 'low';
  scheduledFor: Date;
  sentAt?: Date;
  deliveryStatus: 'pending' | 'sent' | 'delivered' | 'failed';
  relatedSchemeId?: string;
  relatedMilestoneId?: string;
}
```

### Database Schema Design

**User Data (PostgreSQL):**
- Sharded by region for scalability
- Encrypted at rest
- Indexed on userId, location, demographics for fast filtering

**Scheme Data (PostgreSQL + Elasticsearch):**
- PostgreSQL for structured data and relationships
- Elasticsearch for full-text search with personalization
- Read replicas in each region for low latency

**Cache Layer (Redis):**
- User context cache (TTL: 1 hour)
- Scheme data cache (TTL: 24 hours)
- Search results cache (TTL: 1 hour, context-aware keys)
- Session data for anonymous users (TTL: 30 minutes)

**Offline Storage (IndexedDB on client):**
- User profile and preferences
- Saved schemes with full content
- Recent search results
- Pending actions queue

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

