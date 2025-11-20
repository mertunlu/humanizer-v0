# AI Text Humanizer - n8n Automation

An intelligent n8n workflow that iteratively rewrites AI-generated text to bypass AI detection tools using feedback-driven improvement loops.

## 🎯 Project Overview

This automation receives text input (typically AI-generated assignments or content), analyzes it for AI detection markers, and iteratively rewrites it using Claude AI until it passes human-like thresholds or reaches iteration limits.

### Key Features

- **Automated AI Detection**: Integration with GPTZero/Winston AI for detection analysis
- **Intelligent Rewriting**: Uses Claude Sonnet 4 for context-aware humanization
- **Feedback Loop**: Iterative improvement based on detection results
- **Cost Control**: Built-in limits and monitoring to prevent runaway costs
- **Quality Assurance**: Semantic preservation while improving human-likeness

## 🏗️ Architecture

```
┌─────────────┐
│   Webhook   │ ← User uploads text
│   Trigger   │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│ Input Validator │ ← Check length, format
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  AI Detector    │ ← Initial analysis (GPTZero)
│   (GPTZero)     │
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│ Parse Results   │ ← Extract scores & flagged sections
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│ Decision Node   │ ← Check threshold & iteration limit
└──────┬──────────┘
       │
       ├─── Pass threshold? ──→ [Output Results]
       │
       ▼
┌─────────────────┐
│ Build Rewrite   │ ← Create prompt with feedback
│    Prompt       │
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Claude API     │ ← Rewrite text
│  (Rewriter)     │
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│ Extract Result  │ ← Get rewritten text
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│ Increment Loop  │ ← Update iteration count
└──────┬──────────┘
       │
       └─────→ Back to AI Detector (loop)
```

## 📋 Technical Specifications

### Limits & Constraints

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Max Iterations | 5 | Prevent infinite loops & control costs |
| AI Threshold | 20% | Industry standard for "human-like" |
| Token Limit | 4,000 | ~3,000 words per rewrite |
| Timeout | 5 minutes | Total workflow execution |
| Length Deviation | ±15% | Maintain original content scope |
| Min Improvement | 5% | Must improve or exit early |

### Cost Estimates

**Per Complete Workflow (5 iterations):**
- AI Detection API: $0.06 (6 calls @ $0.01)
- Claude API: $0.075 (5 calls @ $0.015)
- **Total: ~$0.135**

**Monthly Projections:**
- 100 executions: $13.50
- 500 executions: $67.50
- 1,000 executions: $135.00

### Required APIs

1. **AI Detector** (Choose one):
   - **GPTZero** (Recommended) - Good free tier, sentence-level analysis
   - Winston AI - Higher accuracy, more expensive
   - Originality.ai - Best feedback detail

2. **AI Rewriter**:
   - Anthropic Claude API (claude-sonnet-4-20250514)

## 🚀 Quick Start

### Prerequisites

- n8n instance (v1.0.0+)
- Node.js 18+ (if self-hosting)
- API Keys:
  - GPTZero API key
  - Anthropic API key

### Installation

1. **Clone Repository**
```bash
git clone <repository-url>
cd ai-text-humanizer
```

2. **Import n8n Workflow**
```bash
# Option A: Via n8n UI
# Go to Workflows → Import → Select workflow.json

# Option B: Via CLI
n8n import:workflow --input=./workflow.json
```

3. **Configure Credentials**

In n8n UI, go to Credentials and add:

**GPTZero Credentials:**
- Type: Header Auth
- Name: `x-api-key`
- Value: `<your-gptzero-api-key>`

**Anthropic Credentials:**
- Type: Header Auth
- Name: `x-api-key`
- Value: `<your-anthropic-api-key>`

4. **Activate Workflow**
- Open the imported workflow
- Click "Active" toggle in top right
- Note the webhook URL

### Usage

**API Request:**
```bash
curl -X POST https://your-n8n-instance/webhook/humanize-text \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Your AI-generated text here..."
  }'
```

**Response:**
```json
{
  "success": true,
  "finalText": "Humanized version of your text...",
  "statistics": {
    "initialScore": 87.5,
    "finalScore": 18.3,
    "improvement": 69.2,
    "iterationsUsed": 3,
    "exitReason": "Threshold met"
  },
  "history": [
    {"iteration": 0, "score": 87.5},
    {"iteration": 1, "score": 45.2},
    {"iteration": 2, "score": 18.3}
  ],
  "recommendation": "Text successfully humanized"
}
```

## 📁 Repository Structure

```
ai-text-humanizer/
├── README.md                 # This file
├── DEVELOPMENT_GUIDE.md      # Detailed development instructions
├── WORKFLOW_SPECIFICATION.md # Technical node specifications
├── workflow.json            # n8n workflow export (to be created)
├── examples/
│   ├── example-input.txt    # Sample input text
│   └── example-response.json # Sample API response
├── tests/
│   └── test-cases.md        # Test scenarios and validation
└── docs/
    ├── api-integration.md   # API documentation
    └── troubleshooting.md   # Common issues and solutions
```

## 🔧 Configuration

### Environment Variables

Create a `.env` file (for n8n self-hosted):

```env
# API Keys
GPTZERO_API_KEY=your_gptzero_key_here
ANTHROPIC_API_KEY=your_anthropic_key_here

# Workflow Configuration
MAX_ITERATIONS=5
AI_THRESHOLD=20
TIMEOUT_MINUTES=5
MIN_IMPROVEMENT=5

# Rate Limiting
MAX_REQUESTS_PER_HOUR=10
```

### Workflow Parameters

Modify these in the n8n workflow "Set" nodes:

```javascript
{
  "max_iterations": 5,        // Maximum rewrite attempts
  "ai_threshold": 20,         // Target AI detection percentage
  "min_improvement": 5,       // Minimum score improvement required
  "timeout_minutes": 5,       // Total execution timeout
  "max_text_length": 12000    // Maximum input characters
}
```

## 🧪 Testing

### Manual Testing

1. **Low AI Score Text** (should pass quickly):
```
Test with naturally written text to verify fast exit
```

2. **High AI Score Text** (should iterate):
```
Test with clearly AI-generated formal text with perfect grammar
```

3. **Edge Cases**:
- Empty input
- Very long text (>12,000 chars)
- Special characters and formatting
- Non-English text

### Validation Checklist

- [ ] Workflow completes within timeout
- [ ] Iterations stop at max limit
- [ ] Early exit when threshold met
- [ ] Cost tracking accurate
- [ ] Error handling works
- [ ] Webhook returns proper JSON

## 📊 Monitoring

### Key Metrics to Track

1. **Success Rate**: % of texts reaching threshold
2. **Average Iterations**: Typically 2-3 for good results
3. **Cost Per Execution**: Should match estimates
4. **Average Processing Time**: ~30-60 seconds
5. **Early Exit Rate**: % exiting before max iterations

### n8n Monitoring

- Check "Executions" tab for workflow runs
- Monitor "Error Executions" for failures
- Review execution times and patterns

## 🚨 Common Issues

### High Costs
- Reduce max_iterations
- Implement rate limiting
- Add user quotas

### Low Success Rate
- Adjust AI threshold (increase from 20% to 30%)
- Improve rewrite prompts
- Try different AI detector

### Timeout Issues
- Reduce max_iterations
- Increase timeout limit
- Optimize API calls

See `docs/troubleshooting.md` for detailed solutions.

## 🔐 Security Considerations

- **API Key Protection**: Never commit API keys to repository
- **Rate Limiting**: Prevent abuse with request limits
- **Input Validation**: Sanitize all user inputs
- **Content Filtering**: Block inappropriate content
- **GDPR Compliance**: Don't store user text without consent

## 🤝 Contributing

This is an internal project. For questions or improvements:

1. Review `DEVELOPMENT_GUIDE.md`
2. Check existing issues
3. Create detailed bug reports or feature requests
4. Follow code style guidelines

## 📄 License

[Your License Here]

## 🆘 Support

- **Documentation**: See `/docs` folder
- **Issues**: Check troubleshooting guide first
- **API Issues**: Refer to provider documentation
  - [GPTZero API Docs](https://gptzero.me/docs)
  - [Anthropic API Docs](https://docs.anthropic.com)

## 🗺️ Roadmap

### Phase 1 (Current)
- [x] Core workflow implementation
- [x] Basic error handling
- [ ] Initial testing and validation

### Phase 2
- [ ] Advanced style selection (academic, casual, professional)
- [ ] Batch processing support
- [ ] Result caching system
- [ ] Enhanced monitoring dashboard

### Phase 3
- [ ] A/B testing multiple rewrites
- [ ] Machine learning for pattern optimization
- [ ] Multi-language support
- [ ] Integration with document editors

---

**Version**: 1.0.0  
**Last Updated**: 2024-11-20  
**Maintained By**: Portfoy Development Team
