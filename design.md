# Design Document: Scheme Navigator for Bharat

## Overview

The Scheme Navigator is a web-based application that helps Indian citizens discover and check eligibility for government schemes. The system consists of three main components:

1. **Frontend Web Application**: A responsive React-based interface that provides user interaction through forms and an AI chat assistant
2. **Backend API Service**: A Node.js/Express server that handles business logic, eligibility checking, and recommendation generation
3. **Scheme Database**: A structured database (PostgreSQL) storing scheme information, eligibility rules, and user profiles

The architecture follows a client-server model with clear separation between presentation, business logic, and data layers. The AI assistant is powered by an LLM API (e.g., OpenAI GPT-4.x or equivalent) that interprets user queries and provides conversational guidance.

### Key Design Decisions

- **Rule-based eligibility engine**: For MVP, we use a deterministic rule engine rather than ML to ensure transparency and accuracy
- **Structured eligibility rules**: Eligibility criteria are stored as JSON objects that can be evaluated programmatically
- **Stateless API**: Backend API is stateless to support horizontal scaling
- **Client-side state management**: React Context API for managing user profile and session state
- **LLM integration**: AI assistant uses function calling to invoke eligibility checks and recommendations

## Architecture

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Frontend (React)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Profile Form │  │ Scheme Search│  │ AI Chat UI   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ HTTPS/REST API
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Backend API (Node.js/Express)             │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │ Eligibility      │  │ Recommendation   │                │
│  │ Checker Service  │  │ Engine Service   │                │
│  └──────────────────┘  └──────────────────┘                │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │ AI Assistant     │  │ Scheme Service   │                │
│  │ Service          │  │                  │                │
│  └──────────────────┘  └──────────────────┘                │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ SQL Queries
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Database (PostgreSQL)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ User Profiles│  │ Schemes      │  │ Eligibility  │      │
│  │              │  │              │  │ Rules        │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ API Calls
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    External Services                         │
│                    (OpenAI API / LLM)                        │
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

**Frontend**:
- Render user interface components
- Manage client-side state (user profile, session)
- Handle form validation and user input
- Display eligibility results and recommendations
- Provide AI chat interface

**Backend API**:
- Expose REST endpoints for all operations
- Validate and sanitize user input
- Execute eligibility checking logic
- Generate scheme recommendations
- Manage AI assistant conversations
- Handle database operations

**Database**:
- Store user profiles securely
- Store scheme information and metadata
- Store eligibility rules in structured format
- Provide efficient querying for recommendations

**External LLM Service**:
- Process natural language queries
- Generate conversational responses
- Invoke backend functions via function calling
- Provide contextual guidance to users

## Components and Interfaces

### 1. Eligibility Checker Service

**Purpose**: Evaluates whether a user meets the eligibility criteria for a specific scheme.

**Interface**:
```typescript
interface EligibilityCheckerService {
  checkEligibility(userId: string, schemeId: string): Promise<EligibilityResult>;
  checkEligibilityWithProfile(profile: UserProfile, schemeId: string): Promise<EligibilityResult>;
}

interface EligibilityResult {
  eligible: boolean;
  matchPercentage: number;
  metCriteria: string[];
  unmetCriteria: CriterionDetail[];
  explanation: string;
}

interface CriterionDetail {
  criterion: string;
  required: any;
  actual: any;
  reason: string;
}
```

**Algorithm**:
1. Fetch scheme eligibility rules from database
2. Fetch user profile data
3. Evaluate each eligibility criterion against user profile
4. For each criterion:
   - Check if user value satisfies the rule (age range, income threshold, location match, etc.)
   - Record whether criterion is met or unmet
5. Calculate match percentage (met criteria / total criteria)
6. Determine overall eligibility (all required criteria must be met)
7. Generate human-readable explanation
8. Return structured result

**Eligibility Rule Format**:
```json
{
  "schemeId": "pm-kisan",
  "rules": [
    {
      "field": "occupation",
      "operator": "equals",
      "value": "farmer",
      "required": true
    },
    {
      "field": "landHolding",
      "operator": "lessThanOrEqual",
      "value": 2,
      "unit": "hectares",
      "required": true
    },
    {
      "field": "age",
      "operator": "between",
      "value": [18, 100],
      "required": true
    }
  ]
}
```

**Supported Operators**:
- `equals`, `notEquals`
- `lessThan`, `lessThanOrEqual`, `greaterThan`, `greaterThanOrEqual`
- `between` (inclusive range)
- `in` (value in list)
- `contains` (for array fields)
- `matches` (regex pattern)

### 2. Recommendation Engine Service

**Purpose**: Generates personalized scheme recommendations based on user profile.

**Interface**:
```typescript
interface RecommendationEngineService {
  getRecommendations(userId: string, options?: RecommendationOptions): Promise<Recommendation[]>;
  getRecommendationsByCategory(userId: string, category: string): Promise<Recommendation[]>;
}

interface RecommendationOptions {
  limit?: number;
  category?: string;
  minMatchPercentage?: number;
}

interface Recommendation {
  scheme: Scheme;
  matchPercentage: number;
  eligibilityStatus: 'eligible' | 'partially_eligible' | 'not_eligible';
  relevanceScore: number;
  reasons: string[];
}
```

**Algorithm**:
1. Fetch all schemes from database
2. For each scheme:
   - Run eligibility check against user profile
   - Calculate match percentage
   - Calculate relevance score based on:
     - Eligibility match percentage (60% weight)
     - Benefit value/impact (20% weight)
     - Scheme popularity (10% weight)
     - User's location match (10% weight)
3. Filter schemes by minimum match percentage (default: 50%)
4. Sort schemes by relevance score (descending)
5. Return top N recommendations (default: 10)

**Relevance Score Calculation**:
```
relevanceScore = (matchPercentage * 0.6) + 
                 (benefitScore * 0.2) + 
                 (popularityScore * 0.1) + 
                 (locationScore * 0.1)
```

### 3. AI Assistant Service

**Purpose**: Provides conversational interface for users to interact with the system.

**Interface**:
```typescript
interface AIAssistantService {
  chat(userId: string, message: string, conversationHistory: Message[]): Promise<AssistantResponse>;
  initializeConversation(userId: string): Promise<ConversationContext>;
}

interface Message {
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: Date;
}

interface AssistantResponse {
  message: string;
  functionCalls?: FunctionCall[];
  suggestedActions?: Action[];
}

interface FunctionCall {
  function: 'checkEligibility' | 'getRecommendations' | 'searchSchemes';
  parameters: Record<string, any>;
  result?: any;
}
```

**Implementation Approach**:
1. Use OpenAI GPT-4 with function calling capability
2. Define functions for:
   - `checkEligibility(schemeId: string)`: Check user's eligibility for a scheme
   - `getRecommendations(category?: string)`: Get personalized recommendations
   - `searchSchemes(query: string)`: Search for schemes by keywords
   - `getSchemeDetails(schemeId: string)`: Get detailed scheme information
3. Maintain conversation context with user profile
4. System prompt includes:
   - Role: "You are a helpful assistant for Indian government schemes"
   - Context: User profile summary
   - Guidelines: Be concise, accurate, and supportive
5. Handle function calls by invoking appropriate backend services
6. Format responses in user-friendly language

### 4. Scheme Service

**Purpose**: Manages scheme data and provides search/retrieval functionality.

**Interface**:
```typescript
interface SchemeService {
  getSchemeById(schemeId: string): Promise<Scheme>;
  searchSchemes(query: string, filters?: SchemeFilters): Promise<Scheme[]>;
  getAllSchemes(filters?: SchemeFilters): Promise<Scheme[]>;
  getSchemesByCategory(category: string): Promise<Scheme[]>;
}

interface SchemeFilters {
  category?: string;
  state?: string;
  benefitType?: string;
  eligibleOnly?: boolean; // Filter by user eligibility
}
```

**Search Implementation**:
- Use PostgreSQL full-text search on scheme name, description, and keywords
- Support fuzzy matching for typos
- Rank results by relevance
- Apply filters after text search

### 5. User Profile Service

**Purpose**: Manages user profile data and validation.

**Interface**:
```typescript
interface UserProfileService {
  createProfile(profile: UserProfileInput): Promise<UserProfile>;
  updateProfile(userId: string, updates: Partial<UserProfileInput>): Promise<UserProfile>;
  getProfile(userId: string): Promise<UserProfile>;
  deleteProfile(userId: string): Promise<void>;
  validateProfile(profile: UserProfileInput): ValidationResult;
}

interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
}

interface ValidationError {
  field: string;
  message: string;
}
```

## Data Models

### User Profile

```typescript
interface UserProfile {
  id: string;
  createdAt: Date;
  updatedAt: Date;
  
  // Demographics
  age: number;
  gender: 'male' | 'female' | 'other';
  state: string;
  district: string;
  pincode: string;
  
  // Economic
  occupation: string;
  annualIncome: number;
  incomeCategory: 'BPL' | 'APL' | 'EWS' | 'LIG' | 'MIG' | 'HIG';
  
  // Family
  familySize: number;
  maritalStatus: 'single' | 'married' | 'widowed' | 'divorced';
  dependents: number;
  
  // Education
  educationLevel: string;
  
  // Additional (optional)
  caste?: string;
  disability?: boolean;
  landHolding?: number;
  rationCardType?: string;
  
  // Metadata
  profileCompleteness: number; // 0-100
}
```

### Scheme

```typescript
interface Scheme {
  id: string;
  name: string;
  nameHindi?: string;
  description: string;
  descriptionHindi?: string;

  > Note (MVP): Hindi fields (`nameHindi`, `descriptionHindi`) are optional and planned for post-MVP. The hackathon MVP will support English only.

  // Classification
  category: string; // 'education', 'healthcare', 'financial', 'agriculture', etc.
  subCategory?: string;
  level: 'central' | 'state' | 'district';
  state?: string; // For state-level schemes
  
  // Benefits
  benefitType: string; // 'monetary', 'subsidy', 'service', 'training', etc.
  benefitAmount?: number;
  benefitDescription: string;
  
  // Eligibility
  eligibilityRules: EligibilityRules;
  eligibilitySummary: string;
  
  // Application
  applicationProcess: string;
  requiredDocuments: string[];
  applicationUrl?: string;
  helplineNumber?: string;
  
  // Metadata
  officialSource: string;
  lastUpdated: Date;
  isActive: boolean;
  popularity: number; // View/application count
  keywords: string[];
}
```

### Eligibility Rules

```typescript
interface EligibilityRules {
  schemeId: string;
  rules: EligibilityRule[];
  logicalOperator: 'AND' | 'OR'; // How to combine rules
}

interface EligibilityRule {
  field: string; // Field name in UserProfile
  operator: RuleOperator;
  value: any; // Expected value or range
  unit?: string; // For numeric values (years, rupees, hectares, etc.)
  required: boolean; // Must be met for eligibility
  description: string; // Human-readable explanation
}

type RuleOperator = 
  | 'equals' 
  | 'notEquals' 
  | 'lessThan' 
  | 'lessThanOrEqual' 
  | 'greaterThan' 
  | 'greaterThanOrEqual' 
  | 'between' 
  | 'in' 
  | 'contains' 
  | 'matches';
```

### Conversation Context

```typescript
interface ConversationContext {
  userId: string;
  conversationId: string;
  messages: Message[];
  userProfile?: UserProfile;
  lastActivity: Date;
}
```

### Database Schema

**users table**:
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  age INTEGER,
  gender VARCHAR(20),
  state VARCHAR(100),
  district VARCHAR(100),
  pincode VARCHAR(10),
  occupation VARCHAR(100),
  annual_income DECIMAL(12, 2),
  income_category VARCHAR(20),
  family_size INTEGER,
  marital_status VARCHAR(20),
  dependents INTEGER,
  education_level VARCHAR(100),
  caste VARCHAR(50),
  disability BOOLEAN,
  land_holding DECIMAL(10, 2),
  ration_card_type VARCHAR(20),
  profile_completeness INTEGER
);
```

**schemes table**:
```sql
CREATE TABLE schemes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(500) NOT NULL,
  name_hindi VARCHAR(500),
  description TEXT NOT NULL,
  description_hindi TEXT,
  category VARCHAR(100) NOT NULL,
  sub_category VARCHAR(100),
  level VARCHAR(20) NOT NULL,
  state VARCHAR(100),
  benefit_type VARCHAR(100),
  benefit_amount DECIMAL(12, 2),
  benefit_description TEXT,
  eligibility_rules JSONB NOT NULL,
  eligibility_summary TEXT,
  application_process TEXT,
  required_documents TEXT[],
  application_url VARCHAR(500),
  helpline_number VARCHAR(20),
  official_source VARCHAR(500),
  last_updated TIMESTAMP DEFAULT NOW(),
  is_active BOOLEAN DEFAULT TRUE,
  popularity INTEGER DEFAULT 0,
  keywords TEXT[]
);

CREATE INDEX idx_schemes_category ON schemes(category);
CREATE INDEX idx_schemes_state ON schemes(state);
CREATE INDEX idx_schemes_keywords ON schemes USING GIN(keywords);
CREATE INDEX idx_schemes_search ON schemes USING GIN(to_tsvector('english', name || ' ' || description));
```

**eligibility_checks table** (for analytics):
```sql
CREATE TABLE eligibility_checks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  scheme_id UUID REFERENCES schemes(id),
  checked_at TIMESTAMP DEFAULT NOW(),
  eligible BOOLEAN,
  match_percentage DECIMAL(5, 2),
  result JSONB
);
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Profile Data Validation

*For any* user profile input, when validation is performed, the system should correctly identify all format and range violations, and accept all valid inputs.

**Validates: Requirements 1.3**

### Property 2: Profile Data Round-Trip

*For any* valid user profile data, saving the profile then retrieving it should return equivalent data with all fields preserved.

**Validates: Requirements 1.4, 1.5**

### Property 3: Incomplete Profile Detection

*For any* user profile missing required fields, the validation should identify exactly which required fields are missing.

**Validates: Requirements 1.6**

### Property 4: Eligibility Evaluation Completeness

*For any* user profile and any scheme, the eligibility checker should evaluate all eligibility rules defined for that scheme and return a determination.

**Validates: Requirements 2.1**

### Property 5: Eligibility Result Completeness

*For any* eligibility check result, it should include a clear determination (eligible/not eligible/partially eligible), match percentage, and explanations for unmet criteria when not eligible, and application information when eligible.

**Validates: Requirements 2.2, 2.3, 2.4**

### Property 6: Missing Profile Data Detection

*For any* scheme with eligibility criteria referencing fields not in the user profile, the system should identify all missing fields required for evaluation.

**Validates: Requirements 2.6**

### Property 7: Recommendation Completeness

*For any* user profile, the recommendation engine should evaluate all active schemes in the database and return results.

**Validates: Requirements 3.1**

### Property 8: Recommendation Ordering

*For any* set of recommendations, they should be ordered by relevance score in descending order (highest relevance first).

**Validates: Requirements 3.2**

### Property 9: Minimum Recommendations Count

*For any* recommendation request, if at least 5 eligible or partially eligible schemes exist, the system should return at least 5 recommendations.

**Validates: Requirements 3.3**

### Property 10: Recommendation Data Completeness

*For any* recommendation in the results, it should include scheme name, description, and eligibility match percentage.

**Validates: Requirements 3.4**

### Property 11: Recommendation Category Filtering

*For any* category filter applied to recommendations, all returned recommendations should belong to that category.

**Validates: Requirements 3.5**

### Property 12: Scheme Data Completeness

*For any* scheme stored in the database, it should include all required fields: name, description, category, eligibility rules, benefits, and application process.

**Validates: Requirements 4.1, 6.1**

### Property 13: Scheme Official Source

*For any* scheme in the database, it should include a valid official source URL.

**Validates: Requirements 4.2**

### Property 14: Scheme Data Round-Trip

*For any* valid scheme data, adding or updating it in the database then retrieving it should return equivalent data with all fields preserved.

**Validates: Requirements 4.4, 6.2, 6.3**

### Property 15: Scheme Language Support

*For any* scheme retrieved from the database, it should have at least an English name and description.

**Validates: Requirements 4.5**

### Property 16: AI Function Invocation

*For any* user query about eligibility, the AI assistant should invoke the eligibility checker function with appropriate parameters.

**Validates: Requirements 5.3**

### Property 17: Conversation Context Preservation

*For any* sequence of messages in a conversation, the conversation context should include all previous messages in chronological order.

**Validates: Requirements 5.6**

### Property 18: Eligibility Rules Format

*For any* eligibility rules stored in the database, they should be parseable as valid JSON and contain all required fields (field, operator, value, required).

**Validates: Requirements 6.4**

### Property 19: Scheme Version History

*For any* scheme update operation, a version history entry should be created with the timestamp and changes.

**Validates: Requirements 6.5**

### Property 20: Scheme Data Validation

*For any* incomplete or invalid scheme data, validation should reject it and identify all missing or invalid fields.

**Validates: Requirements 6.6**

### Property 21: Search Result Relevance

*For any* search query, all returned schemes should match the query in at least one searchable field (name, description, keywords, category, or benefits).

**Validates: Requirements 7.1, 7.2**

### Property 22: Search Eligibility Filtering

*For any* eligibility filter applied to search results, all returned schemes should match the user's eligibility status according to that filter.

**Validates: Requirements 7.5**

### Property 23: Profile Deletion Completeness

*For any* user profile, after deletion, attempting to retrieve that profile should fail, and no personal data should remain in the database.

**Validates: Requirements 8.3**

### Property 24: Data Access Audit Logging

*For any* operation that accesses user profile data, an audit log entry should be created with timestamp, operation type, and user identifier.

**Validates: Requirements 8.6**

## Error Handling

### Error Categories

**Validation Errors**:
- Invalid user input (age out of range, invalid pincode format, etc.)
- Incomplete profile data
- Invalid scheme data
- Malformed eligibility rules

**Data Errors**:
- User profile not found
- Scheme not found
- Database connection failures
- Data corruption

**External Service Errors**:
- LLM API failures (timeout, rate limit, service unavailable)
- Network errors

**Business Logic Errors**:
- Cannot evaluate eligibility due to missing profile data
- No schemes available for recommendations
- Search query too broad or too narrow

### Error Handling Strategy

**Validation Errors**:
- Return 400 Bad Request with detailed error messages
- Include field-level errors for forms
- Provide suggestions for correction

**Data Errors**:
- Return 404 Not Found for missing resources
- Return 500 Internal Server Error for database failures
- Log errors for debugging
- Implement retry logic for transient failures

**External Service Errors**:
- Implement exponential backoff for retries
- Provide fallback responses when LLM is unavailable
- Cache LLM responses when possible
- Return 503 Service Unavailable when external services are down

**Business Logic Errors**:
- Return 422 Unprocessable Entity with explanation
- Provide actionable guidance to users
- Suggest alternative actions

### Error Response Format

```typescript
interface ErrorResponse {
  error: {
    code: string;
    message: string;
    details?: Record<string, any>;
    suggestions?: string[];
  };
}
```

### Graceful Degradation

- If LLM service is unavailable, provide form-based interface as fallback
- If recommendation engine fails, fall back to showing popular schemes
- If eligibility checker fails, provide manual eligibility criteria for user review
- Cache frequently accessed data to reduce database load

## Testing Strategy

### Dual Testing Approach

The system will use both unit tests and property-based tests to ensure comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs

Both testing approaches are complementary and necessary. Unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across a wide range of inputs.

### Property-Based Testing

**Framework**: We will use **fast-check** (for TypeScript/JavaScript) as our property-based testing library.

**Configuration**:
- Each property test will run a minimum of 100 iterations
- Each test will be tagged with a comment referencing the design property
- Tag format: `// Feature: scheme-navigator, Property {number}: {property_text}`

**Property Test Coverage**:
Each correctness property listed above will be implemented as a single property-based test. The tests will generate random valid inputs and verify that the properties hold across all generated cases.

**Example Property Test Structure**:
```typescript
// Feature: scheme-navigator, Property 2: Profile Data Round-Trip
test('profile data round-trip preserves all fields', async () => {
  await fc.assert(
    fc.asyncProperty(
      userProfileArbitrary, // Generator for random user profiles
      async (profile) => {
        const saved = await profileService.createProfile(profile);
        const retrieved = await profileService.getProfile(saved.id);
        expect(retrieved).toEqual(saved);
      }
    ),
    { numRuns: 100 }
  );
});
```

### Unit Testing

**Framework**: Jest for TypeScript/JavaScript

**Unit Test Focus**:
- Specific examples demonstrating correct behavior
- Edge cases (empty inputs, boundary values, null/undefined handling)
- Error conditions and error messages
- Integration points between components
- Mock external dependencies (LLM API, database)

**Test Coverage Goals**:
- 80%+ code coverage for business logic
- 100% coverage for eligibility rule evaluation
- 100% coverage for validation logic

### Integration Testing

**Scope**:
- End-to-end API testing
- Database integration
- LLM integration (with mocked responses)

**Tools**:
- Supertest for API testing
- Test database with seed data

### Test Data

**Seed Data**:
- 20+ representative government schemes covering different categories
- Sample user profiles representing different demographics
- Edge case schemes with complex eligibility rules

**Generators for Property Tests**:
- User profile generator (valid and invalid profiles)
- Scheme generator (valid schemes with various eligibility rules)
- Eligibility rule generator (all operator types)
- Search query generator

### Testing Priorities for MVP

Given the 2-month timeline, prioritize:
1. Property tests for eligibility checking (core functionality)
2. Basic unit tests for recommendation scoring logic
3. Unit tests for validation logic
4. Unit tests for error handling
5. Integration tests for critical API endpoints
6. Manual testing for AI assistant behavior

> Note (MVP): For the hackathon, we will prioritize property-based tests for the eligibility engine and basic unit tests for APIs. Full property-based testing coverage for all components is planned post-MVP.


### Continuous Testing

- Run unit tests on every commit
- Run property tests on every pull request
- Run integration tests before deployment
- Monitor test execution time and optimize slow tests
