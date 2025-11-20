# Development Guide - AI Text Humanizer

This guide provides detailed, step-by-step instructions for building the AI Text Humanizer n8n workflow. Follow this guide sequentially for successful implementation.

## 🎯 Development Phases

### Phase 1: Environment Setup
### Phase 2: Core Workflow Construction
### Phase 3: Integration & Testing
### Phase 4: Optimization & Deployment

---

## Phase 1: Environment Setup

### 1.1 n8n Installation

**Self-Hosted Option:**
```bash
# Using Docker (Recommended)
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

# Access at: http://localhost:5678
```

**Cloud Option:**
- Sign up at https://n8n.io
- Create new workspace
- Note your instance URL

### 1.2 API Key Acquisition

**GPTZero API:**
1. Visit https://gptzero.me
2. Sign up for account
3. Navigate to API section
4. Generate API key
5. Note the key format: `sk-...`

**Anthropic API:**
1. Visit https://console.anthropic.com
2. Create account or sign in
3. Go to API Keys section
4. Create new key
5. Copy the key (starts with `sk-ant-api03-...`)

### 1.3 Credential Configuration in n8n

**Add GPTZero Credentials:**
```
1. Click Settings (gear icon) → Credentials
2. Click "Add Credential"
3. Search for "Header Auth"
4. Configure:
   - Credential Name: "GPTZero API"
   - Name: x-api-key
   - Value: [paste your GPTZero API key]
5. Save
```

**Add Anthropic Credentials:**
```
1. Credentials → Add Credential
2. Search for "Header Auth"
3. Configure:
   - Credential Name: "Anthropic API"
   - Name: x-api-key
   - Value: [paste your Anthropic API key]
4. Add additional header:
   - Name: anthropic-version
   - Value: 2023-06-01
5. Save
```

---

## Phase 2: Core Workflow Construction

### 2.1 Create New Workflow

1. Click "+ New Workflow"
2. Name it: "AI Text Humanizer v1.0"
3. Save workflow

### 2.2 Node-by-Node Implementation

#### Node 1: Webhook Trigger

**Purpose**: Receive text input from external sources

**Configuration:**
```
Node Type: Webhook
HTTP Method: POST
Path: humanize-text
Response Mode: Last Node
Response Code: 200
```

**Expected Input:**
```json
{
  "text": "The text to humanize goes here..."
}
```

**Implementation Steps:**
1. Drag "Webhook" node to canvas
2. Set path to `humanize-text`
3. Set method to POST
4. Enable "Wait for Webhook Call"
5. Test webhook URL appears

---

#### Node 2: Input Validation

**Purpose**: Validate input format and length

**Configuration:**
```
Node Type: Function
Function Name: validateInput
```

**Code:**
```javascript
// Validate Input Node
const inputData = $input.item.json;

// Extract text from various possible input formats
let text = inputData.text || inputData.body?.text || inputData.query?.text;

// Validation checks
if (!text) {
  throw new Error("Missing required field: 'text'. Please provide text to humanize.");
}

// Remove excessive whitespace
text = text.trim();

if (text.length === 0) {
  throw new Error("Text cannot be empty after trimming whitespace.");
}

if (text.length > 12000) {
  throw new Error(`Text too long: ${text.length} characters. Maximum allowed: 12,000 characters.`);
}

if (text.length < 50) {
  throw new Error("Text too short. Minimum length: 50 characters for meaningful analysis.");
}

// Initialize workflow state
return [{
  json: {
    originalText: text,
    currentText: text,
    textLength: text.length,
    wordCount: text.split(/\s+/).length,
    iterationCount: 0,
    maxIterations: 5,
    aiThreshold: 20,
    minImprovement: 5,
    history: [],
    timestamp: new Date().toISOString()
  }
}];
```

**Validation Rules:**
- Text must exist
- Length: 50 - 12,000 characters
- Trims whitespace
- Initializes workflow state

---

#### Node 3: AI Detector (Initial)

**Purpose**: Get initial AI detection score

**Configuration:**
```
Node Type: HTTP Request
Method: POST
URL: https://api.gptzero.me/v2/predict/text
Authentication: Existing Credentials (GPTZero API)
```

**Headers:**
```json
{
  "Content-Type": "application/json",
  "x-api-key": "={{$credentials.gptzero.apiKey}}"
}
```

**Body:**
```json
{
  "document": "={{$json.currentText}}",
  "version": "2024-01-09"
}
```

**Implementation Steps:**
1. Add HTTP Request node
2. Connect to validation node
3. Select GPTZero credentials
4. Configure URL and method
5. Set JSON body with document field
6. Test with sample text

**Expected Response Structure:**
```json
{
  "documents": [{
    "completely_generated_prob": 0.87,
    "average_generated_prob": 0.82,
    "sentences": [
      {
        "sentence": "Example sentence.",
        "generated_prob": 0.95,
        "index": 0
      }
    ]
  }]
}
```

---

#### Node 4: Parse Detection Results

**Purpose**: Extract and structure detection feedback

**Configuration:**
```
Node Type: Function
Function Name: parseDetectionResults
```

**Code:**
```javascript
// Parse AI Detection Results
const previousData = $('validateInput').item.json;
const detectionResponse = $input.item.json;

// Extract detection metrics
const document = detectionResponse.documents[0];
const aiProbability = Math.round(document.completely_generated_prob * 100);
const avgProbability = Math.round(document.average_generated_prob * 100);
const sentences = document.sentences || [];

// Identify problematic sentences (>70% AI probability)
const flaggedSentences = sentences
  .filter(s => s.generated_prob > 0.7)
  .map(s => ({
    text: s.sentence.substring(0, 100), // Limit length for prompt
    score: Math.round(s.generated_prob * 100),
    index: s.index
  }))
  .slice(0, 10); // Limit to top 10 for prompt efficiency

// Generate specific feedback
const feedback = generateFeedback(aiProbability, flaggedSentences);

// Calculate if threshold is met
const passesThreshold = aiProbability < previousData.aiThreshold;

// Store current iteration results
const iterationResult = {
  iteration: previousData.iterationCount,
  aiScore: aiProbability,
  avgScore: avgProbability,
  flaggedCount: flaggedSentences.length,
  timestamp: new Date().toISOString()
};

function generateFeedback(score, flagged) {
  const suggestions = [];
  
  if (score > 80) {
    suggestions.push("Text shows strong AI patterns - needs significant humanization");
    suggestions.push("Add personal anecdotes or specific real-world examples");
    suggestions.push("Vary sentence structure dramatically");
    suggestions.push("Include casual language and contractions");
  } else if (score > 50) {
    suggestions.push("Moderate AI detection - refine specific sections");
    suggestions.push("Focus on flagged sentences with high AI probability");
    suggestions.push("Add transitional phrases that feel more natural");
  } else if (score > 20) {
    suggestions.push("Close to threshold - minor adjustments needed");
    suggestions.push("Vary word choice in remaining flagged sections");
    suggestions.push("Add subtle imperfections in grammar or style");
  }
  
  if (flagged.length > 5) {
    suggestions.push(`High number of flagged sentences (${flagged.length}) - systematic rewrite needed`);
  }
  
  return {
    score: score,
    level: score > 80 ? 'high' : score > 50 ? 'medium' : 'low',
    suggestions: suggestions
  };
}

// Merge all data
return [{
  json: {
    ...previousData,
    currentAiScore: aiProbability,
    avgAiScore: avgProbability,
    passesThreshold: passesThreshold,
    flaggedSentences: flaggedSentences,
    feedback: feedback,
    currentIterationResult: iterationResult,
    history: [...previousData.history, iterationResult]
  }
}];
```

---

#### Node 5: Decision Node (IF)

**Purpose**: Decide whether to continue iterating or exit

**Configuration:**
```
Node Type: IF
Conditions: Multiple conditions with AND/OR logic
```

**Condition Logic:**
```javascript
// Continue Loop IF:
// 1. AI score is above threshold AND
// 2. Haven't reached max iterations AND
// 3. Not stuck (no improvement check)

// Condition 1: Above threshold
{{$json.currentAiScore >= $json.aiThreshold}}

// Condition 2: Below max iterations
{{$json.iterationCount < $json.maxIterations}}

// Condition 3: Check for improvement (if past iteration 1)
{{
  $json.iterationCount < 2 || 
  ($json.history[$json.history.length - 2].aiScore - $json.currentAiScore >= 3)
}}
```

**Implementation:**
1. Add IF node
2. Configure "true" path → continue to rewriter
3. Configure "false" path → go to final output
4. Add all three conditions
5. Set to "Match all conditions (AND)"

**Exit Scenarios:**
- ✅ Threshold met (score < 20%)
- ✅ Max iterations reached (5 attempts)
- ✅ No improvement detected (stuck)

---

#### Node 6: Build Rewrite Prompt

**Purpose**: Create contextual prompt for Claude with feedback

**Configuration:**
```
Node Type: Function
Function Name: buildRewritePrompt
```

**Code:**
```javascript
// Build Rewrite Prompt for Claude
const data = $input.item.json;

// Determine intensity of rewriting based on score
const intensity = data.currentAiScore > 80 ? 'aggressive' : 
                 data.currentAiScore > 50 ? 'moderate' : 'gentle';

// Build context about flagged sections
const flaggedContext = data.flaggedSentences.length > 0 
  ? `\n\nMOST PROBLEMATIC SENTENCES (focus here):\n${
      data.flaggedSentences
        .slice(0, 5)
        .map((s, i) => `${i + 1}. "${s.text}" (AI Score: ${s.score}%)`)
        .join('\n')
    }`
  : '';

// Iteration-specific guidance
const iterationGuidance = getIterationGuidance(data.iterationCount, data.history);

const prompt = `You are an expert at rewriting text to sound natural and human-written while preserving meaning.

CURRENT STATUS:
- Iteration: ${data.iterationCount + 1}/${data.maxIterations}
- Current AI Detection Score: ${data.currentAiScore}%
- Target Score: <${data.aiThreshold}%
- Improvement Needed: ${data.currentAiScore - data.aiThreshold}%

TEXT TO REWRITE:
${data.currentText}
${flaggedContext}

SPECIFIC FEEDBACK:
${data.feedback.suggestions.map(s => `- ${s}`).join('\n')}

${iterationGuidance}

REWRITING INSTRUCTIONS (${intensity} approach):
${getRewritingInstructions(intensity)}

CRITICAL RULES:
- Maintain the core meaning and key information
- Keep length within ±15% of original (${Math.round(data.textLength * 0.85)}-${Math.round(data.textLength * 1.15)} characters)
- Make text sound like a real person wrote it naturally
- DO NOT use phrases like "in conclusion", "in summary", "furthermore" excessively
- DO NOT make every sentence perfect - humans make small mistakes
- DO NOT over-explain or add unnecessary elaboration

Rewrite the text now, focusing on natural human expression:`;

function getIterationGuidance(iteration, history) {
  if (iteration === 0) {
    return "FIRST ATTEMPT: Be bold in your changes - significant humanization needed.";
  }
  
  const previousScore = history[history.length - 2]?.aiScore;
  const improvement = previousScore - history[history.length - 1]?.aiScore;
  
  if (improvement < 5) {
    return `PREVIOUS ATTEMPT HAD MINIMAL IMPROVEMENT (${improvement}%). Try a completely different approach - vary your strategy significantly.`;
  }
  
  return `GOOD PROGRESS: Previous iteration improved score by ${improvement}%. Continue this approach with refinements.`;
}

function getRewritingInstructions(intensity) {
  const instructions = {
    aggressive: `- Completely restructure sentences, don't just rephrase
- Break up long, complex sentences into shorter, varied ones
- Add personal touches: "I think", "in my experience", "I've noticed"
- Use contractions extensively: "don't", "it's", "I've"
- Include conversational elements: "Well,", "You know,", "Actually,"
- Vary paragraph lengths dramatically
- Add occasional minor grammatical quirks (but keep it readable)`,
    
    moderate: `- Restructure obvious AI patterns while keeping good elements
- Vary sentence beginnings and structures
- Add natural transitions and connectors
- Use some contractions and casual language
- Include specific examples or elaborations where appropriate
- Mix formal and informal tone naturally`,
    
    gentle: `- Focus on the specifically flagged sentences
- Adjust word choice in remaining problematic areas
- Add subtle variations in sentence rhythm
- Include minor stylistic imperfections
- Refine transitions to feel more natural`
  };
  
  return instructions[intensity];
}

return [{
  json: {
    ...data,
    rewritePrompt: prompt,
    rewriteIntensity: intensity
  }
}];
```

**Prompt Strategy:**
- Adaptive intensity based on AI score
- Iteration-specific guidance
- Focus on flagged sentences
- Preserve meaning while humanizing

---

#### Node 7: Claude API (Rewriter)

**Purpose**: Send text to Claude for humanization

**Configuration:**
```
Node Type: HTTP Request
Method: POST
URL: https://api.anthropic.com/v1/messages
Authentication: Existing Credentials (Anthropic API)
```

**Headers:**
```json
{
  "Content-Type": "application/json",
  "x-api-key": "={{$credentials.anthropic.apiKey}}",
  "anthropic-version": "2023-06-01"
}
```

**Body:**
```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 4000,
  "temperature": 0.8,
  "messages": [
    {
      "role": "user",
      "content": "={{$json.rewritePrompt}}"
    }
  ]
}
```

**Important Notes:**
- Temperature 0.8 for natural variation
- Max tokens 4000 for longer texts
- Using latest Claude Sonnet model

---

#### Node 8: Extract Rewritten Text

**Purpose**: Parse Claude's response and update state

**Configuration:**
```
Node Type: Function
Function Name: extractRewrittenText
```

**Code:**
```javascript
// Extract Rewritten Text from Claude Response
const previousData = $('buildRewritePrompt').item.json;
const claudeResponse = $input.item.json;

// Extract text from Claude response
const rewrittenText = claudeResponse.content[0].text.trim();

// Calculate text metrics
const originalLength = previousData.textLength;
const newLength = rewrittenText.length;
const lengthDeviation = Math.abs((newLength - originalLength) / originalLength * 100);

// Validate length deviation
if (lengthDeviation > 20) {
  console.log(`WARNING: Length deviation is ${lengthDeviation.toFixed(1)}% (threshold: 20%)`);
}

// Update state for next iteration
return [{
  json: {
    ...previousData,
    previousText: previousData.currentText,
    currentText: rewrittenText,
    textLength: newLength,
    iterationCount: previousData.iterationCount + 1,
    lengthDeviation: lengthDeviation,
    rewriteTimestamp: new Date().toISOString()
  }
}];
```

---

#### Node 9: Loop Back to Detection

**Purpose**: Connect back to AI Detector for re-analysis

**Implementation:**
1. Connect Node 8 output to Node 3 (AI Detector) input
2. This creates the feedback loop
3. The IF node (Node 5) will control when to exit

**Loop Flow:**
```
Extract Text → AI Detector → Parse Results → Decision Node
                                                    ↓
                                                  IF true
                                                    ↓
                                            Build Prompt → Claude → Extract → [Loop Back]
```

---

#### Node 10: Final Output Formatter

**Purpose**: Format results for API response

**Configuration:**
```
Node Type: Function
Function Name: formatFinalOutput
Connected to: IF node "false" output
```

**Code:**
```javascript
// Format Final Output
const data = $input.item.json;

// Determine exit reason
let exitReason;
if (data.passesThreshold) {
  exitReason = "Threshold met - text successfully humanized";
} else if (data.iterationCount >= data.maxIterations) {
  exitReason = "Maximum iterations reached";
} else {
  exitReason = "No significant improvement detected";
}

// Calculate overall statistics
const initialScore = data.history[0]?.aiScore || 100;
const finalScore = data.currentAiScore;
const totalImprovement = initialScore - finalScore;
const improvementPercentage = ((totalImprovement / initialScore) * 100).toFixed(1);

// Build response
const response = {
  success: data.passesThreshold,
  finalText: data.currentText,
  
  statistics: {
    initialAiScore: initialScore,
    finalAiScore: finalScore,
    improvement: totalImprovement,
    improvementPercentage: `${improvementPercentage}%`,
    iterationsUsed: data.iterationCount,
    exitReason: exitReason,
    processingTime: calculateProcessingTime(data.timestamp)
  },
  
  iterationHistory: data.history.map(h => ({
    iteration: h.iteration,
    aiScore: h.aiScore,
    flaggedSentences: h.flaggedCount
  })),
  
  textMetrics: {
    originalLength: data.originalText.length,
    finalLength: data.currentText.length,
    lengthChange: `${((data.currentText.length - data.originalText.length) / data.originalText.length * 100).toFixed(1)}%`,
    originalWords: data.originalText.split(/\s+/).length,
    finalWords: data.currentText.split(/\s+/).length
  },
  
  recommendation: getRecommendation(data.passesThreshold, finalScore),
  
  metadata: {
    workflowVersion: "1.0.0",
    timestamp: new Date().toISOString(),
    model: "claude-sonnet-4-20250514"
  }
};

function calculateProcessingTime(startTimestamp) {
  const start = new Date(startTimestamp);
  const end = new Date();
  const seconds = Math.round((end - start) / 1000);
  return `${seconds} seconds`;
}

function getRecommendation(success, score) {
  if (success) {
    return "✅ Text successfully humanized and ready to use";
  } else if (score < 30) {
    return "⚠️ Close to threshold - consider one manual review";
  } else if (score < 50) {
    return "⚠️ Moderate AI detection - manual editing recommended";
  } else {
    return "❌ High AI detection remains - significant manual rewrite needed";
  }
}

return [{
  json: response
}];
```

---

#### Node 11: Respond to Webhook

**Purpose**: Send formatted response back to caller

**Configuration:**
```
Node Type: Respond to Webhook
Response Code: 200
```

**Implementation:**
1. Add "Respond to Webhook" node
2. Connect from Final Output Formatter
3. Set response code to 200
4. Data automatically sent as JSON

---

## Phase 3: Error Handling

### Add Error Handling Nodes

**Error Catcher for API Failures:**

```javascript
// Node Type: Error Trigger
// Triggered on: This workflow

// Error Handler Function
const error = $input.item.json;

const errorResponse = {
  success: false,
  error: {
    message: error.message || "An unexpected error occurred",
    type: error.name || "WorkflowError",
    timestamp: new Date().toISOString()
  },
  recommendation: "Please check your input and try again. If the issue persists, contact support."
};

// Log error for debugging
console.error("Workflow Error:", error);

return [{ json: errorResponse }];
```

**Add Timeout Protection:**

Add to validation node:
```javascript
// Set timeout metadata
$execution.customData.startTime = Date.now();
$execution.customData.timeoutMs = 300000; // 5 minutes

// Check in each iteration
if (Date.now() - $execution.customData.startTime > $execution.customData.timeoutMs) {
  throw new Error("Workflow timeout exceeded (5 minutes)");
}
```

---

## Phase 4: Testing & Validation

### Test Cases

**Test 1: Low AI Score (Quick Exit)**
```json
{
  "text": "I went to the store yesterday and, you know, it was pretty busy. The lines were long but I managed to grab what I needed. Funny enough, I ran into my old friend Sarah there - hadn't seen her in years! We ended up chatting for like 30 minutes in the parking lot."
}
```
Expected: 1-2 iterations, success

**Test 2: High AI Score (Full Iterations)**
```json
{
  "text": "The implementation of artificial intelligence in modern business environments has demonstrated substantial potential for optimizing operational efficiency. Organizations that leverage machine learning algorithms can significantly enhance their decision-making processes. Furthermore, the integration of automated systems facilitates streamlined workflows and reduces human error."
}
```
Expected: 4-5 iterations, significant improvement

**Test 3: Edge Cases**
- Empty text
- Very short text (< 50 chars)
- Very long text (> 12,000 chars)
- Special characters
- Non-English text

### Validation Checklist

- [ ] All nodes connected properly
- [ ] Credentials configured correctly
- [ ] Loop logic functions without infinite loops
- [ ] Error handling catches all failures
- [ ] Response format matches specification
- [ ] Cost tracking is accurate
- [ ] Processing time is reasonable (< 2 minutes typically)

---

## Phase 5: Optimization

### Performance Improvements

**Caching Strategy:**
```javascript
// Add to validation node
const textHash = require('crypto')
  .createHash('md5')
  .update($json.text)
  .digest('hex');

// Check cache (implement with Redis or similar)
const cachedResult = await checkCache(textHash);
if (cachedResult) {
  return [{ json: cachedResult }];
}
```

**Parallel Processing:**
- Consider splitting very long texts
- Process paragraphs independently
- Merge results at the end

**Cost Optimization:**
- Implement early exit when close to threshold
- Use cheaper detector for quick pre-check
- Cache detection results for 24 hours

---

## Deployment Checklist

### Pre-Deployment
- [ ] All test cases pass
- [ ] Error handling tested
- [ ] Credentials secured
- [ ] Rate limiting configured
- [ ] Monitoring set up
- [ ] Documentation complete

### Deployment Steps
1. Export workflow JSON
2. Backup current production
3. Import new workflow
4. Test in production with low traffic
5. Monitor for errors
6. Gradually increase traffic
7. Document any issues

### Post-Deployment
- [ ] Monitor execution times
- [ ] Track success rates
- [ ] Review costs daily
- [ ] Collect user feedback
- [ ] Iterate on improvements

---

## Troubleshooting Guide

### Common Issues

**Issue: Infinite Loop**
- Check IF node conditions
- Verify max iterations enforced
- Add emergency timeout

**Issue: High Costs**
- Review iteration counts
- Check for stuck workflows
- Implement user quotas

**Issue: Poor Results**
- Adjust rewrite prompts
- Try different AI detector
- Increase max iterations

**Issue: API Timeouts**
- Reduce max tokens
- Optimize prompt length
- Add retry logic

---

## Maintenance

### Regular Tasks

**Daily:**
- Check error logs
- Monitor costs
- Review success rates

**Weekly:**
- Analyze common failure patterns
- Update prompts based on results
- Review API usage trends

**Monthly:**
- Evaluate new AI detectors
- Test with updated Claude models
- Optimize based on usage patterns

---

## Next Steps

1. Complete Phase 2 (build all nodes)
2. Test thoroughly with all test cases
3. Deploy to staging environment
4. Monitor and iterate
5. Deploy to production
6. Document lessons learned

For questions or issues during development, refer to:
- n8n documentation: https://docs.n8n.io
- GPTZero API docs: https://gptzero.me/docs
- Anthropic API docs: https://docs.anthropic.com

---

**Document Version**: 1.0.0  
**Last Updated**: 2024-11-20  
**Maintained By**: Portfoy Development Team
