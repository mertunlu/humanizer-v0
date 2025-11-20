# Workflow Specification - AI Text Humanizer

This document contains the complete technical specification for the AI Text Humanizer n8n workflow, including data structures, API contracts, and implementation details.

## Table of Contents

1. [Data Flow Architecture](#data-flow-architecture)
2. [API Specifications](#api-specifications)
3. [Node Specifications](#node-specifications)
4. [Data Structures](#data-structures)
5. [State Management](#state-management)
6. [Error Handling](#error-handling)
7. [Performance Requirements](#performance-requirements)

---

## Data Flow Architecture

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                          ENTRY POINT                            │
│                    Webhook: POST /humanize-text                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      INPUT VALIDATION                           │
│  • Check text exists                                            │
│  • Validate length (50-12,000 chars)                           │
│  • Initialize workflow state                                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   AI DETECTION (GPTZero)                        │
│  • Send text for analysis                                       │
│  • Get AI probability scores                                    │
│  • Identify flagged sentences                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   PARSE DETECTION RESULTS                       │
│  • Extract AI score percentage                                  │
│  • Identify problematic sections                                │
│  • Generate rewrite feedback                                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DECISION GATE (IF)                         │
│  Conditions:                                                     │
│  • AI Score >= Threshold (20%)                                  │
│  • Iterations < Max (5)                                         │
│  • Improvement detected (>3% per iteration)                     │
└──────────────┬──────────────────────────────────┬───────────────┘
               │ FALSE                            │ TRUE
               │ (Exit)                           │ (Continue)
               ▼                                  ▼
┌──────────────────────────┐    ┌───────────────────────────────┐
│   FORMAT FINAL OUTPUT    │    │    BUILD REWRITE PROMPT       │
│  • Success/failure       │    │  • Contextual instructions    │
│  • Statistics            │    │  • Flagged sentences          │
│  • History               │    │  • Iteration guidance         │
│  • Recommendation        │    └───────────────┬───────────────┘
└──────────┬───────────────┘                    │
           │                                    ▼
           │              ┌─────────────────────────────────────┐
           │              │      CLAUDE API (Rewriter)          │
           │              │  • Process rewrite instructions     │
           │              │  • Generate humanized text          │
           │              │  • Maintain meaning                 │
           │              └──────────────┬──────────────────────┘
           │                             │
           │                             ▼
           │              ┌─────────────────────────────────────┐
           │              │    EXTRACT REWRITTEN TEXT           │
           │              │  • Parse Claude response            │
           │              │  • Update iteration count           │
           │              │  • Validate length deviation        │
           │              └──────────────┬──────────────────────┘
           │                             │
           │                             └──────────┐
           │                                        │
           │    ┌───────────────────────────────────┘
           │    │ LOOP BACK TO AI DETECTION
           │    │
           ▼    ▼
┌─────────────────────────────────────────────────────────────────┐
│                     RESPOND TO WEBHOOK                          │
│  Return JSON response with results                              │
└─────────────────────────────────────────────────────────────────┘
```

### State Transitions

```
[INIT] → [VALIDATING] → [DETECTING] → [PARSING]
                                          ↓
                                    [EVALUATING]
                                     ↓       ↓
                            [REWRITING]   [COMPLETE]
                                 ↓
                            [DETECTING] (loop back)
```

---

## API Specifications

### 1. Webhook Input API

**Endpoint**: `POST /webhook/humanize-text`

**Request Headers**:
```http
Content-Type: application/json
```

**Request Body**:
```json
{
  "text": "string (required, 50-12000 characters)"
}
```

**Success Response** (200 OK):
```json
{
  "success": true,
  "finalText": "string",
  "statistics": {
    "initialAiScore": 87.5,
    "finalAiScore": 18.3,
    "improvement": 69.2,
    "improvementPercentage": "79.1%",
    "iterationsUsed": 3,
    "exitReason": "Threshold met - text successfully humanized",
    "processingTime": "45 seconds"
  },
  "iterationHistory": [
    {
      "iteration": 0,
      "aiScore": 87.5,
      "flaggedSentences": 12
    },
    {
      "iteration": 1,
      "aiScore": 54.2,
      "flaggedSentences": 7
    },
    {
      "iteration": 2,
      "aiScore": 18.3,
      "flaggedSentences": 2
    }
  ],
  "textMetrics": {
    "originalLength": 1250,
    "finalLength": 1312,
    "lengthChange": "+5.0%",
    "originalWords": 210,
    "finalWords": 218
  },
  "recommendation": "✅ Text successfully humanized and ready to use",
  "metadata": {
    "workflowVersion": "1.0.0",
    "timestamp": "2024-11-20T10:30:45.123Z",
    "model": "claude-sonnet-4-20250514"
  }
}
```

**Error Response** (400 Bad Request):
```json
{
  "success": false,
  "error": {
    "message": "Text too long: 15000 characters. Maximum allowed: 12,000 characters.",
    "type": "ValidationError",
    "timestamp": "2024-11-20T10:30:45.123Z"
  },
  "recommendation": "Please check your input and try again. If the issue persists, contact support."
}
```

**Error Response** (500 Internal Server Error):
```json
{
  "success": false,
  "error": {
    "message": "API request failed: Rate limit exceeded",
    "type": "APIError",
    "timestamp": "2024-11-20T10:30:45.123Z"
  },
  "recommendation": "Please try again in a few moments."
}
```

### 2. GPTZero API Integration

**Endpoint**: `POST https://api.gptzero.me/v2/predict/text`

**Request Headers**:
```http
Content-Type: application/json
x-api-key: sk-[your-api-key]
```

**Request Body**:
```json
{
  "document": "string (the text to analyze)",
  "version": "2024-01-09"
}
```

**Response Structure**:
```json
{
  "documents": [
    {
      "completely_generated_prob": 0.875,
      "average_generated_prob": 0.823,
      "sentences": [
        {
          "sentence": "Example sentence text.",
          "generated_prob": 0.95,
          "index": 0
        }
      ],
      "paragraphs": [
        {
          "completely_generated_prob": 0.88,
          "num_sentences": 3,
          "start_sentence_index": 0,
          "end_sentence_index": 2
        }
      ]
    }
  ],
  "class_probabilities": {
    "ai": 0.875,
    "human": 0.125,
    "mixed": 0.000
  }
}
```

**Rate Limits**:
- Free tier: 50 requests/day
- Paid tier: 10,000 requests/month
- Response time: 2-5 seconds typical

### 3. Anthropic Claude API Integration

**Endpoint**: `POST https://api.anthropic.com/v1/messages`

**Request Headers**:
```http
Content-Type: application/json
x-api-key: sk-ant-api03-[your-api-key]
anthropic-version: 2023-06-01
```

**Request Body**:
```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 4000,
  "temperature": 0.8,
  "messages": [
    {
      "role": "user",
      "content": "string (the rewrite prompt)"
    }
  ]
}
```

**Response Structure**:
```json
{
  "id": "msg_01XYZ...",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "The rewritten text..."
    }
  ],
  "model": "claude-sonnet-4-20250514",
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 1250,
    "output_tokens": 890
  }
}
```

**Pricing**:
- Input: $3.00 per million tokens
- Output: $15.00 per million tokens
- Typical cost per rewrite: $0.01-0.02

---

## Node Specifications

### Node 1: Webhook Trigger

**Type**: `n8n-nodes-base.webhook`

**Configuration**:
```json
{
  "httpMethod": "POST",
  "path": "humanize-text",
  "responseMode": "lastNode",
  "responseCode": 200,
  "options": {}
}
```

**Output Schema**:
```json
{
  "headers": {
    "content-type": "application/json",
    "user-agent": "string"
  },
  "body": {
    "text": "string"
  },
  "query": {}
}
```

### Node 2: Input Validation

**Type**: `n8n-nodes-base.function`

**Input Schema**: Webhook output

**Output Schema**:
```json
{
  "originalText": "string",
  "currentText": "string",
  "textLength": 0,
  "wordCount": 0,
  "iterationCount": 0,
  "maxIterations": 5,
  "aiThreshold": 20,
  "minImprovement": 5,
  "history": [],
  "timestamp": "ISO8601 string"
}
```

**Error Conditions**:
- Missing text field → Error 400
- Empty text → Error 400
- Text < 50 chars → Error 400
- Text > 12,000 chars → Error 400

### Node 3: AI Detector (HTTP Request)

**Type**: `n8n-nodes-base.httpRequest`

**Configuration**:
```json
{
  "method": "POST",
  "url": "https://api.gptzero.me/v2/predict/text",
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "httpHeaderAuth",
  "sendHeaders": true,
  "headerParameters": {
    "parameter": [
      {
        "name": "x-api-key",
        "value": "={{$credentials.gptzero.apiKey}}"
      }
    ]
  },
  "sendBody": true,
  "bodyParameters": {
    "parameter": [
      {
        "name": "document",
        "value": "={{$json.currentText}}"
      },
      {
        "name": "version",
        "value": "2024-01-09"
      }
    ]
  },
  "options": {
    "timeout": 30000
  }
}
```

**Expected Response Time**: 2-5 seconds

**Retry Policy**:
- Max retries: 2
- Backoff: Exponential (2s, 4s)

### Node 4: Parse Detection Results

**Type**: `n8n-nodes-base.function`

**Input Schema**: GPTZero API response + previous state

**Processing Logic**:
1. Extract `completely_generated_prob` and convert to percentage
2. Filter sentences with `generated_prob > 0.7`
3. Generate feedback based on score ranges
4. Add iteration result to history
5. Merge with previous state

**Output Schema**:
```json
{
  "...previousState": {},
  "currentAiScore": 0,
  "avgAiScore": 0,
  "passesThreshold": false,
  "flaggedSentences": [
    {
      "text": "string",
      "score": 0,
      "index": 0
    }
  ],
  "feedback": {
    "score": 0,
    "level": "high|medium|low",
    "suggestions": ["string"]
  },
  "currentIterationResult": {
    "iteration": 0,
    "aiScore": 0,
    "avgScore": 0,
    "flaggedCount": 0,
    "timestamp": "ISO8601 string"
  },
  "history": [{}]
}
```

### Node 5: Decision Node (IF)

**Type**: `n8n-nodes-base.if`

**Configuration**:
```json
{
  "conditions": {
    "combinator": "and",
    "conditions": [
      {
        "leftValue": "={{$json.currentAiScore}}",
        "rightValue": "={{$json.aiThreshold}}",
        "operator": {
          "type": "number",
          "operation": "gte"
        }
      },
      {
        "leftValue": "={{$json.iterationCount}}",
        "rightValue": "={{$json.maxIterations}}",
        "operator": {
          "type": "number",
          "operation": "lt"
        }
      },
      {
        "leftValue": "={{$json.iterationCount < 2 || ($json.history[$json.history.length - 2].aiScore - $json.currentAiScore >= 3)}}",
        "rightValue": true,
        "operator": {
          "type": "boolean",
          "operation": "equals"
        }
      }
    ]
  }
}
```

**Truth Table**:

| AI Score | Iterations | Improvement | Result |
|----------|------------|-------------|--------|
| ≥20% | <5 | ≥3% | Continue (true) |
| <20% | Any | Any | Exit (false) |
| Any | ≥5 | Any | Exit (false) |
| ≥20% | <5 | <3% | Exit (false) |

### Node 6: Build Rewrite Prompt

**Type**: `n8n-nodes-base.function`

**Input Schema**: Previous state with detection results

**Processing Logic**:
1. Determine rewrite intensity (aggressive/moderate/gentle)
2. Build flagged sentences context
3. Generate iteration-specific guidance
4. Construct full prompt
5. Add metadata

**Output Schema**:
```json
{
  "...previousState": {},
  "rewritePrompt": "string (full prompt)",
  "rewriteIntensity": "aggressive|moderate|gentle"
}
```

**Prompt Template Structure**:
```
[STATUS SECTION]
- Iteration info
- Current scores
- Improvement needed

[TEXT TO REWRITE]
- Full current text
- Flagged sections highlighted

[FEEDBACK SECTION]
- Specific suggestions
- Iteration guidance

[INSTRUCTIONS SECTION]
- Rewriting approach
- Rules and constraints

[CRITICAL RULES]
- Preserve meaning
- Length constraints
- Natural expression
```

### Node 7: Claude API (Rewriter)

**Type**: `n8n-nodes-base.httpRequest`

**Configuration**:
```json
{
  "method": "POST",
  "url": "https://api.anthropic.com/v1/messages",
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "httpHeaderAuth",
  "sendHeaders": true,
  "headerParameters": {
    "parameter": [
      {
        "name": "x-api-key",
        "value": "={{$credentials.anthropic.apiKey}}"
      },
      {
        "name": "anthropic-version",
        "value": "2023-06-01"
      }
    ]
  },
  "sendBody": true,
  "specifyBody": "json",
  "jsonBody": "={{ JSON.stringify({\n  model: 'claude-sonnet-4-20250514',\n  max_tokens: 4000,\n  temperature: 0.8,\n  messages: [{\n    role: 'user',\n    content: $json.rewritePrompt\n  }]\n}) }}",
  "options": {
    "timeout": 60000
  }
}
```

**Expected Response Time**: 10-30 seconds

**Token Usage Estimates**:
- Input: 1,000-2,000 tokens
- Output: 800-1,500 tokens
- Total cost: $0.01-0.02 per call

### Node 8: Extract Rewritten Text

**Type**: `n8n-nodes-base.function`

**Input Schema**: Claude API response + previous state

**Processing Logic**:
1. Extract text from `content[0].text`
2. Calculate length metrics
3. Validate length deviation (<20%)
4. Increment iteration count
5. Store previous text version

**Output Schema**:
```json
{
  "...previousState": {},
  "previousText": "string",
  "currentText": "string (rewritten)",
  "textLength": 0,
  "iterationCount": 0,
  "lengthDeviation": 0,
  "rewriteTimestamp": "ISO8601 string"
}
```

### Node 9: Final Output Formatter

**Type**: `n8n-nodes-base.function`

**Input Schema**: Complete workflow state

**Output Schema**: See [Webhook Output API](#1-webhook-input-api)

### Node 10: Respond to Webhook

**Type**: `n8n-nodes-base.respondToWebhook`

**Configuration**:
```json
{
  "respondWith": "allEntries",
  "responseCode": 200,
  "options": {}
}
```

---

## Data Structures

### Workflow State Object

Complete state maintained throughout workflow:

```typescript
interface WorkflowState {
  // Original input
  originalText: string;
  currentText: string;
  textLength: number;
  wordCount: number;
  
  // Configuration
  maxIterations: number;        // 5
  aiThreshold: number;          // 20
  minImprovement: number;       // 5
  
  // Current iteration state
  iterationCount: number;
  currentAiScore: number;
  avgAiScore: number;
  passesThreshold: boolean;
  
  // Detection results
  flaggedSentences: Array<{
    text: string;
    score: number;
    index: number;
  }>;
  
  // Feedback
  feedback: {
    score: number;
    level: 'high' | 'medium' | 'low';
    suggestions: string[];
  };
  
  // Iteration history
  history: Array<{
    iteration: number;
    aiScore: number;
    avgScore: number;
    flaggedCount: number;
    timestamp: string;
  }>;
  
  // Metadata
  timestamp: string;
  previousText?: string;
  lengthDeviation?: number;
  rewriteTimestamp?: string;
  rewritePrompt?: string;
  rewriteIntensity?: 'aggressive' | 'moderate' | 'gentle';
}
```

### Iteration Result Object

```typescript
interface IterationResult {
  iteration: number;
  aiScore: number;
  avgScore: number;
  flaggedCount: number;
  timestamp: string;
  textSnapshot?: string;  // Optional, for debugging
}
```

### Flagged Sentence Object

```typescript
interface FlaggedSentence {
  text: string;           // Max 100 chars for prompt efficiency
  score: number;          // 0-100
  index: number;          // Position in original text
  context?: string;       // Surrounding sentences for context
}
```

---

## State Management

### State Initialization

Occurs in Input Validation node:
```javascript
{
  originalText: input.text,
  currentText: input.text,
  textLength: input.text.length,
  wordCount: input.text.split(/\s+/).length,
  iterationCount: 0,
  maxIterations: 5,
  aiThreshold: 20,
  minImprovement: 5,
  history: [],
  timestamp: new Date().toISOString()
}
```

### State Updates

**After Detection**:
- Add `currentAiScore`
- Add `flaggedSentences`
- Add `feedback`
- Append to `history`

**After Rewrite**:
- Update `currentText`
- Store `previousText`
- Increment `iterationCount`
- Calculate `lengthDeviation`

### State Persistence

State is maintained in memory throughout workflow execution. No external storage required.

**State Size Estimates**:
- Minimum: ~2KB
- Typical: ~5-10KB
- Maximum: ~50KB (with full history)

---

## Error Handling

### Error Types

**1. Validation Errors (400)**:
```typescript
{
  code: 'VALIDATION_ERROR',
  message: 'Text too long: 15000 characters. Maximum: 12,000',
  field: 'text',
  value: '[text preview...]'
}
```

**2. API Errors (502)**:
```typescript
{
  code: 'API_ERROR',
  message: 'GPTZero API request failed',
  provider: 'gptzero',
  statusCode: 429,
  retryAfter: 60
}
```

**3. Timeout Errors (504)**:
```typescript
{
  code: 'TIMEOUT_ERROR',
  message: 'Workflow exceeded 5 minute timeout',
  elapsedTime: 305,
  maxTime: 300
}
```

**4. Rate Limit Errors (429)**:
```typescript
{
  code: 'RATE_LIMIT_ERROR',
  message: 'API rate limit exceeded',
  provider: 'anthropic',
  resetTime: '2024-11-20T11:00:00Z'
}
```

### Error Recovery Strategies

**Retry Logic**:
```javascript
{
  maxRetries: 2,
  retryDelay: [2000, 4000],  // Exponential backoff
  retryableErrors: [429, 500, 502, 503, 504]
}
```

**Fallback Behavior**:
1. Primary detector fails → Use fallback detector
2. Rewriter fails → Return best iteration so far
3. Complete failure → Return original text with error

### Error Logging

```javascript
{
  timestamp: 'ISO8601',
  errorType: 'string',
  errorMessage: 'string',
  stackTrace: 'string',
  workflowState: {
    iteration: 0,
    aiScore: 0,
    textLength: 0
  },
  metadata: {
    userId: 'string',
    executionId: 'string'
  }
}
```

---

## Performance Requirements

### Response Time Targets

| Scenario | Target | Maximum |
|----------|--------|---------|
| 1 iteration | 15s | 30s |
| 3 iterations | 45s | 90s |
| 5 iterations | 90s | 180s |
| Full workflow | 120s | 300s |

### Throughput Targets

- **Concurrent executions**: 10
- **Requests per minute**: 20
- **Daily capacity**: 1,000 executions

### Resource Limits

**Per Execution**:
- Memory: 512MB
- CPU: 1 vCPU
- Network: 50MB data transfer
- Storage: N/A (stateless)

**Per Node**:
- Timeout: 60s (API calls)
- Timeout: 10s (Function nodes)

### Optimization Strategies

**1. Early Exit**:
```javascript
// Exit if improvement is minimal after 2 iterations
if (iterationCount >= 2 && improvement < minImprovement) {
  exitEarly = true;
}
```

**2. Prompt Optimization**:
- Limit flagged sentences to top 5
- Truncate long sentences to 100 chars
- Remove redundant instructions

**3. Caching** (Future):
```javascript
// Cache detection results for 1 hour
const cacheKey = hash(text);
const cached = await cache.get(cacheKey);
if (cached) return cached;
```

---

## Monitoring & Metrics

### Key Performance Indicators

**Success Metrics**:
- Success rate (% reaching threshold)
- Average iterations to success
- Average AI score improvement
- User satisfaction

**Performance Metrics**:
- Average execution time
- 95th percentile execution time
- API call latency
- Error rate

**Cost Metrics**:
- Cost per execution
- API costs by provider
- Monthly burn rate
- Cost per successful humanization

### Monitoring Implementation

```javascript
// Add to each major node
const metrics = {
  nodeName: 'AI Detector',
  startTime: Date.now(),
  endTime: null,
  duration: null,
  success: true,
  error: null
};

try {
  // Node logic
  metrics.endTime = Date.now();
  metrics.duration = metrics.endTime - metrics.startTime;
} catch (error) {
  metrics.success = false;
  metrics.error = error.message;
  throw error;
} finally {
  logMetrics(metrics);
}
```

---

## Security Considerations

### API Key Management

**Storage**:
- Use n8n credentials system
- Never log or expose keys
- Rotate keys quarterly

**Access Control**:
- Restrict webhook to authorized sources
- Implement API key authentication for webhook
- Rate limit by IP address

### Input Sanitization

```javascript
// Remove potentially malicious content
function sanitizeInput(text) {
  // Remove script tags
  text = text.replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '');
  
  // Limit special characters
  text = text.replace(/[^\w\s\.,!?;:()\-'"]/g, '');
  
  // Normalize whitespace
  text = text.replace(/\s+/g, ' ').trim();
  
  return text;
}
```

### Data Privacy

**KVKK Compliance** (Turkish GDPR):
- Don't store input text beyond execution
- Log only metadata, not content
- Allow users to delete execution history
- Encrypt data in transit (HTTPS)

### Rate Limiting

```javascript
{
  globalLimit: {
    requests: 100,
    window: '1h'
  },
  perUserLimit: {
    requests: 10,
    window: '1h'
  },
  perIPLimit: {
    requests: 50,
    window: '1h'
  }
}
```

---

## Testing Specifications

### Unit Test Cases

**Input Validation**:
- ✓ Valid text (500 chars)
- ✓ Minimum length (50 chars)
- ✓ Maximum length (12,000 chars)
- ✗ Empty text
- ✗ Too short (<50 chars)
- ✗ Too long (>12,000 chars)
- ✗ Missing text field

**Detection Parsing**:
- ✓ High AI score (>80%)
- ✓ Medium AI score (40-80%)
- ✓ Low AI score (<20%)
- ✓ Many flagged sentences (>10)
- ✓ Few flagged sentences (<3)
- ✓ Zero flagged sentences

**Decision Logic**:
- ✓ Below threshold → exit
- ✓ Max iterations → exit
- ✓ No improvement → exit
- ✓ Above threshold + iterations left → continue

### Integration Test Cases

**Complete Workflows**:
1. Quick success (1-2 iterations)
2. Full iterations (5 attempts)
3. Marginal improvement (stuck case)
4. API failure recovery
5. Timeout handling

### Load Test Specifications

**Test Scenarios**:
- 10 concurrent users
- 50 requests in 1 minute
- 1,000 requests in 1 day
- Sustained load (100 req/hour)

**Success Criteria**:
- 95% success rate
- <2% error rate
- P95 latency <180s
- No memory leaks

---

## Deployment Specifications

### Environment Variables

```bash
# Required
GPTZERO_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-api03-...

# Optional (with defaults)
MAX_ITERATIONS=5
AI_THRESHOLD=20
MIN_IMPROVEMENT=5
TIMEOUT_MINUTES=5
MAX_TEXT_LENGTH=12000
RATE_LIMIT_REQUESTS_PER_HOUR=10
```

### Health Check Endpoint

```javascript
// GET /health
{
  status: 'healthy',
  version: '1.0.0',
  uptime: 12345,
  checks: {
    gptzero: 'connected',
    anthropic: 'connected',
    memory: '45MB / 512MB',
    activeExecutions: 3
  }
}
```

### Rollback Procedure

1. Export current workflow JSON
2. Backup to `workflows/backup/`
3. Test new version in staging
4. Deploy to production
5. Monitor for 1 hour
6. Rollback if error rate >5%

---

## Appendix

### A. Glossary

- **AI Detection Score**: Percentage likelihood text is AI-generated (0-100%)
- **Iteration**: One complete cycle of detect → rewrite
- **Flagged Sentence**: Sentence with AI probability >70%
- **Threshold**: Target AI score for successful humanization (20%)
- **Early Exit**: Stopping iterations before max due to success or lack of improvement

### B. References

- [n8n Documentation](https://docs.n8n.io)
- [GPTZero API Docs](https://gptzero.me/docs)
- [Anthropic API Docs](https://docs.anthropic.com)
- [KVKK Compliance Guide](https://www.kvkk.gov.tr)

### C. Version History

- v1.0.0 (2024-11-20): Initial specification

---

**Document Version**: 1.0.0  
**Last Updated**: 2024-11-20  
**Maintained By**: Portfoy Development Team  
**Status**: Production Ready
