# Design Document: ContentFlow AI

## Overview

ContentFlow AI is a serverless social media content generation platform that transforms long-form content into platform-optimized posts for Twitter, LinkedIn, Instagram, and Facebook. The system uses Claude Sonnet 4 for intelligent content transformation and Unsplash API for relevant image selection.

The architecture follows a stateless, serverless design pattern with a React frontend and AWS Lambda backend. This MVP focuses on core content generation functionality without user authentication or data persistence.

### Key Design Decisions

1. **Serverless Architecture**: AWS Lambda + API Gateway for scalability and cost-effectiveness
2. **Stateless Design**: No database required for MVP, simplifying deployment and maintenance
3. **AI-First Approach**: Claude Sonnet 4 provides high-quality, context-aware content transformation
4. **Single API Call**: All 4 posts generated in one request to minimize latency
5. **Concurrent Processing**: Parallel API calls to Claude and Unsplash to optimize response time

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────┐
│                         Frontend                            │
│                  (React + Vite + Tailwind)                  │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Input Form   │  │ Loading UI   │  │ Results Grid │       │
│  │ - Validation │  │ - Progress   │  │ - 4 Posts    │       │
│  │ - Char Count │  │ - Status     │  │ - Copy Btns  │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ HTTPS
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway                            │
│                   (REST API Endpoint)                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      AWS Lambda                             │
│                  (Content Generator)                        │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Request Handler                                      │   │
│  │  1. Validate input                                   │   │
│  │  2. Call Claude API (parallel for 4 platforms)       │   │
│  │  3. Call Unsplash API (parallel for 4 images)        │   │
│  │  4. Aggregate results                                │   │
│  │  5. Return response                                  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                    │                    │
                    │                    │
                    ▼                    ▼
        ┌──────────────────┐  ┌──────────────────┐
        │   Claude API     │  │  Unsplash API    │
        │  (Sonnet 4)      │  │  (Stock Images)  │
        └──────────────────┘  └──────────────────┘
```

### Data Flow

1. User enters content (300-10,000 chars) in Frontend
2. Frontend validates input and sends POST request to API Gateway
3. API Gateway triggers Lambda function
4. Lambda validates request and initiates parallel processing:
   - 4 concurrent calls to Claude API (one per platform)
   - 4 concurrent calls to Unsplash API (one per platform theme)
5. Lambda aggregates results into structured response
6. Response returns through API Gateway to Frontend
7. Frontend displays 4 platform-specific posts with images

## Components and Interfaces

### Frontend Components

#### InputForm Component
```typescript
interface InputFormProps {
  onSubmit: (content: string) => void;
  isLoading: boolean;
}

interface InputFormState {
  content: string;
  charCount: number;
  isValid: boolean;
  errorMessage: string | null;
}

// Responsibilities:
// - Display textarea with character counter
// - Validate content length (300-10,000 chars)
// - Show validation errors
// - Disable submit during loading
// - Clear form after successful submission
```

#### LoadingIndicator Component
```typescript
interface LoadingIndicatorProps {
  status: string;
  progress: number;
}

// Responsibilities:
// - Display animated loading spinner
// - Show current status message
// - Display progress percentage
// - Provide visual feedback during 30-60s wait
```

#### ResultsGrid Component
```typescript
interface Post {
  platform: 'twitter' | 'linkedin' | 'instagram' | 'facebook';
  content: string;
  imageUrl: string;
  hashtags: string[];
}

interface ResultsGridProps {
  posts: Post[];
  onCopy: (post: Post) => void;
}

// Responsibilities:
// - Display 4 posts in responsive grid
// - Show platform icon/label for each post
// - Render images with proper aspect ratios
// - Provide copy-to-clipboard functionality
// - Handle empty/error states
```

### Backend Components

#### Lambda Handler
```python
def lambda_handler(event, context):
    """
    Main entry point for content generation requests.
    
    Input:
    {
        "content": string (300-10,000 chars)
    }
    
    Output:
    {
        "posts": [
            {
                "platform": string,
                "content": string,
                "imageUrl": string,
                "hashtags": string[]
            }
        ]
    }
    
    Errors:
    {
        "error": string,
        "message": string
    }
    """
    pass
```

#### Content Generator
```python
class ContentGenerator:
    """
    Generates platform-specific content using Claude API.
    """
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.client = anthropic.Anthropic(api_key=api_key)
    
    def generate_post(self, content: str, platform: str) -> dict:
        """
        Generate a single platform-optimized post.
        
        Args:
            content: Original long-form content
            platform: Target platform (twitter/linkedin/instagram/facebook)
        
        Returns:
            {
                "content": string,
                "hashtags": string[]
            }
        """
        pass
    
    def generate_all_posts(self, content: str) -> list:
        """
        Generate posts for all 4 platforms concurrently.
        
        Uses ThreadPoolExecutor for parallel API calls.
        """
        pass
```

#### Image Service
```python
class ImageService:
    """
    Retrieves relevant images from Unsplash API.
    """
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.unsplash.com"
    
    def get_image(self, query: str, platform: str) -> str:
        """
        Get a relevant image URL for the content.
        
        Args:
            query: Search query derived from content
            platform: Target platform for aspect ratio optimization
        
        Returns:
            Image URL string or default placeholder
        """
        pass
    
    def get_all_images(self, queries: list) -> list:
        """
        Get images for all platforms concurrently.
        """
        pass
```

#### Platform Optimizer
```python
class PlatformOptimizer:
    """
    Defines platform-specific constraints and prompts.
    """
    
    PLATFORM_SPECS = {
        'twitter': {
            'max_chars': 280,
            'tone': 'concise and punchy',
            'hashtag_count': (2, 3),
            'style': 'conversational'
        },
        'linkedin': {
            'word_range': (150, 300),
            'tone': 'professional and insightful',
            'hashtag_count': (3, 5),
            'style': 'thought leadership'
        },
        'instagram': {
            'tone': 'engaging and visual',
            'hashtag_count': (5, 8),
            'style': 'storytelling with emojis'
        },
        'facebook': {
            'tone': 'conversational and community-focused',
            'hashtag_count': (2, 4),
            'style': 'friendly and inclusive'
        }
    }
    
    def get_prompt(self, content: str, platform: str) -> str:
        """
        Generate Claude API prompt for specific platform.
        """
        pass
    
    def validate_post(self, post: str, platform: str) -> bool:
        """
        Validate post meets platform constraints.
        """
        pass
```

## Data Models

### Request Model
```typescript
interface GenerateRequest {
  content: string;  // 300-10,000 characters
}
```

### Response Model
```typescript
interface GenerateResponse {
  posts: Post[];
  generatedAt: string;  // ISO 8601 timestamp
}

interface Post {
  platform: Platform;
  content: string;
  imageUrl: string;
  hashtags: string[];
  metadata: PostMetadata;
}

type Platform = 'twitter' | 'linkedin' | 'instagram' | 'facebook';

interface PostMetadata {
  charCount?: number;      // For Twitter
  wordCount?: number;      // For LinkedIn
  estimatedReach?: string; // Optional engagement hint
}
```

### Error Model
```typescript
interface ErrorResponse {
  error: ErrorType;
  message: string;
  details?: string;
}

type ErrorType = 
  | 'VALIDATION_ERROR'
  | 'AI_API_ERROR'
  | 'IMAGE_API_ERROR'
  | 'TIMEOUT_ERROR'
  | 'RATE_LIMIT_ERROR'
  | 'INTERNAL_ERROR';
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Input Validation Bounds
*For any* input string, the validation function should accept it if and only if its length is between 300 and 10,000 characters (inclusive).
**Validates: Requirements 1.1, 1.2, 1.3**

### Property 2: Character Count Accuracy
*For any* input string, the character count function should return the exact length of the string.
**Validates: Requirements 1.4**

### Property 3: Four Platform Output
*For any* valid input content, the content generator should produce exactly 4 posts, one for each platform (Twitter, LinkedIn, Instagram, Facebook).
**Validates: Requirements 2.1**

### Property 4: Twitter Character Limit
*For any* generated Twitter post, the content length should not exceed 280 characters.
**Validates: Requirements 2.2**

### Property 5: LinkedIn Word Count Range
*For any* generated LinkedIn post, the word count should be between 150 and 300 words (inclusive).
**Validates: Requirements 2.3**

### Property 6: Post Uniqueness
*For any* set of generated posts from the same input, no two posts should be identical, and no post should be a simple substring truncation of the original input.
**Validates: Requirements 3.4**

### Property 7: Image URL Presence
*For any* generated post, it should include a valid image URL (either from Unsplash or a placeholder).
**Validates: Requirements 4.1, 4.4**

### Property 8: Platform-Specific Aspect Ratios
*For any* platform, the image rendering function should apply the correct aspect ratio style for that platform.
**Validates: Requirements 4.5**

### Property 9: Hashtag Count Range
*For any* generated post, it should include between 3 and 5 hashtags (inclusive).
**Validates: Requirements 5.1**

### Property 10: Response Time Limit
*For any* valid input content, the backend should return all 4 generated posts within 60 seconds.
**Validates: Requirements 6.1**

### Property 11: Progress Updates During Processing
*For any* content generation request, the backend should emit at least one progress update before returning the final response.
**Validates: Requirements 6.2**

### Property 12: Loading Indicator Display
*For any* generation request in progress, the frontend should display a loading indicator with the current status.
**Validates: Requirements 6.3**

### Property 13: Error Logging and User Messages
*For any* error that occurs, the system should both log the error details and display a user-friendly message.
**Validates: Requirements 7.5**

### Property 14: Stateless Request Processing
*For any* two consecutive requests with the same input, the backend should produce independent results without sharing state between requests.
**Validates: Requirements 8.1, 8.2, 8.4**

### Property 15: Complete Single Response
*For any* valid request, the backend should return a complete response containing all 4 posts with all required fields (platform, content, imageUrl, hashtags) in a single API call.
**Validates: Requirements 8.5**

### Property 16: All Posts Rendered
*For any* successful generation response, the frontend should render all 4 posts from the response.
**Validates: Requirements 9.2**

### Property 17: Platform Labels Present
*For any* rendered post, it should display a label indicating its target platform.
**Validates: Requirements 9.3**

### Property 18: Copy to Clipboard Functionality
*For any* post, the copy function should successfully copy the post content to the clipboard.
**Validates: Requirements 9.4**

### Property 19: API Key Confidentiality
*For any* response or log output, it should not contain API key values or patterns that could expose credentials.
**Validates: Requirements 10.2**

### Property 20: Authentication Headers Present
*For any* external API call, the request should include the required authentication headers.
**Validates: Requirements 10.3**

### Property 21: Input Validation Before API Calls
*For any* request, input validation should complete successfully before any external API calls are made.
**Validates: Requirements 10.5**

## Error Handling

### Error Categories

1. **Validation Errors**
   - Input too short (< 300 chars)
   - Input too long (> 10,000 chars)
   - Missing required fields
   - Invalid characters or encoding

2. **External API Errors**
   - Claude API failures (rate limits, timeouts, service errors)
   - Unsplash API failures (rate limits, network errors)
   - Authentication failures

3. **Timeout Errors**
   - Generation exceeds 60-second limit
   - External API calls timeout

4. **Internal Errors**
   - Unexpected exceptions
   - Data processing errors
   - Memory or resource constraints

### Error Handling Strategy

#### Frontend Error Handling
```typescript
interface ErrorHandler {
  handleValidationError(error: ValidationError): void;
  handleNetworkError(error: NetworkError): void;
  handleTimeoutError(error: TimeoutError): void;
  handleUnknownError(error: Error): void;
}

// Error Display Strategy:
// - Show user-friendly message in UI
// - Provide actionable guidance (e.g., "Reduce content length")
// - Offer retry option for transient errors
// - Log full error details to console for debugging
```

#### Backend Error Handling
```python
class ErrorHandler:
    """
    Centralized error handling for Lambda function.
    """
    
    def handle_validation_error(self, error: ValidationError) -> dict:
        """Return 400 with validation details."""
        return {
            'statusCode': 400,
            'body': json.dumps({
                'error': 'VALIDATION_ERROR',
                'message': str(error),
                'details': error.details
            })
        }
    
    def handle_api_error(self, error: APIError) -> dict:
        """Handle external API failures with graceful degradation."""
        # For image errors: use placeholder
        # For AI errors: return error to user
        pass
    
    def handle_timeout_error(self) -> dict:
        """Return 504 with timeout message."""
        return {
            'statusCode': 504,
            'body': json.dumps({
                'error': 'TIMEOUT_ERROR',
                'message': 'Content generation took too long. Please try again with shorter content.'
            })
        }
```

### Graceful Degradation

1. **Image Service Failure**: Continue generation with placeholder images
2. **Partial Generation Failure**: Return successfully generated posts with error details for failed ones
3. **Rate Limit Handling**: Return clear message with retry-after information

## Testing Strategy

### Dual Testing Approach

This project requires both unit tests and property-based tests to ensure comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs

Together, these approaches provide comprehensive coverage where unit tests catch concrete bugs and property tests verify general correctness.

### Property-Based Testing

We will use **fast-check** (for TypeScript/JavaScript) and **Hypothesis** (for Python) as our property-based testing libraries.

**Configuration Requirements:**
- Each property test must run a minimum of 100 iterations
- Each test must be tagged with a comment referencing its design property
- Tag format: `// Feature: contentflow-ai, Property {number}: {property_text}`

**Property Test Examples:**

```typescript
// Feature: contentflow-ai, Property 1: Input Validation Bounds
test('validates input length bounds', () => {
  fc.assert(
    fc.property(fc.string(), (input) => {
      const isValid = validateInput(input);
      const length = input.length;
      
      if (length >= 300 && length <= 10000) {
        expect(isValid).toBe(true);
      } else {
        expect(isValid).toBe(false);
      }
    }),
    { numRuns: 100 }
  );
});

// Feature: contentflow-ai, Property 4: Twitter Character Limit
test('Twitter posts never exceed 280 characters', () => {
  fc.assert(
    fc.property(
      fc.string({ minLength: 300, maxLength: 10000 }),
      async (input) => {
        const posts = await generatePosts(input);
        const twitterPost = posts.find(p => p.platform === 'twitter');
        
        expect(twitterPost.content.length).toBeLessThanOrEqual(280);
      }
    ),
    { numRuns: 100 }
  );
});
```

```python
# Feature: contentflow-ai, Property 3: Four Platform Output
@given(st.text(min_size=300, max_size=10000))
@settings(max_examples=100)
def test_generates_four_platform_posts(content):
    posts = generate_posts(content)
    
    assert len(posts) == 4
    platforms = {post['platform'] for post in posts}
    assert platforms == {'twitter', 'linkedin', 'instagram', 'facebook'}
```

### Unit Testing Strategy

**Frontend Unit Tests:**
- Input validation logic
- Character counting
- Copy to clipboard functionality
- Error message display
- Component rendering with specific props

**Backend Unit Tests:**
- Request parsing and validation
- Platform-specific prompt generation
- Response formatting
- Error handling for specific scenarios
- Mock API responses

**Integration Tests:**
- End-to-end flow with mocked external APIs
- Error propagation from backend to frontend
- Timeout handling
- Graceful degradation scenarios

### Test Coverage Goals

- **Unit Test Coverage**: 80%+ for business logic
- **Property Test Coverage**: All 21 correctness properties implemented
- **Integration Test Coverage**: All critical user flows
- **Error Handling Coverage**: All error types and edge cases

### Testing Tools

**Frontend:**
- Vitest for unit and property tests
- React Testing Library for component tests
- fast-check for property-based testing
- MSW (Mock Service Worker) for API mocking

**Backend:**
- pytest for unit tests
- Hypothesis for property-based testing
- moto for AWS service mocking
- responses for HTTP mocking

### Continuous Testing

- Run unit tests on every commit
- Run property tests (100 iterations) on every PR
- Run integration tests before deployment
- Monitor test execution time (property tests may take longer)

## Implementation Notes

### Technology Stack

**Frontend:**
- React 18+ with TypeScript
- Vite for build tooling
- Tailwind CSS for styling
- Axios for HTTP requests

**Backend:**
- Python 3.11+ runtime
- AWS Lambda with 512MB memory, 60s timeout
- anthropic Python SDK for Claude API
- requests library for Unsplash API

**Infrastructure:**
- AWS API Gateway (REST API)
- AWS Lambda (single function)
- Environment variables for API keys
- CORS enabled for frontend domain

### Deployment Considerations

1. **Environment Variables:**
   - `CLAUDE_API_KEY`: Anthropic API key
   - `UNSPLASH_ACCESS_KEY`: Unsplash API access key
   - `CORS_ORIGIN`: Allowed frontend origin

2. **Lambda Configuration:**
   - Memory: 512MB (sufficient for concurrent API calls)
   - Timeout: 60 seconds (matches requirement)
   - Concurrency: 10 (for MVP, can scale up)

3. **API Gateway Configuration:**
   - Enable CORS
   - Request validation enabled
   - Rate limiting: 100 requests/minute per IP

4. **Cost Optimization:**
   - Lambda: Pay per invocation (~$0.20 per 1M requests)
   - Claude API: Pay per token (~$3 per 1M tokens)
   - Unsplash: Free tier (50 requests/hour)

### Security Considerations

1. **API Key Management:**
   - Store in AWS Secrets Manager or environment variables
   - Never commit to version control
   - Rotate keys regularly

2. **Input Sanitization:**
   - Validate content length
   - Strip potentially harmful characters
   - Prevent injection attacks

3. **Rate Limiting:**
   - Implement at API Gateway level
   - Track per-IP request counts
   - Return 429 for exceeded limits

4. **CORS Configuration:**
   - Whitelist specific frontend domain
   - Don't use wildcard (*) in production

### Performance Optimization

1. **Concurrent API Calls:**
   - Use ThreadPoolExecutor for parallel Claude calls
   - Use asyncio for parallel Unsplash calls
   - Aggregate results efficiently

2. **Response Caching:**
   - Consider caching identical requests (future enhancement)
   - Cache Unsplash images by query

3. **Cold Start Mitigation:**
   - Keep Lambda warm with scheduled pings
   - Optimize import statements
   - Use Lambda layers for dependencies

### Future Enhancements (Out of Scope for MVP)

1. User authentication and saved posts
2. Custom platform templates
3. Scheduling and direct posting to platforms
4. Analytics and engagement tracking
5. Team collaboration features
6. Custom image uploads
7. A/B testing for post variations
