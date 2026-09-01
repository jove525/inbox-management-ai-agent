# Inbox Management AI Agent v2.0

> An intelligent email triage system powered by Claude AI with confidence thresholds, sentiment analysis, and human-in-the-loop approvals.

## Overview

Automatically triage, categorize, and respond to emails using Claude 3.5 Sonnet with built-in safety features, context awareness, and interactive Slack notifications. Built for n8n workflow automation platform.

### Key Features

- **Claude AI Integration** - Advanced sentiment analysis and natural language understanding
- **Confidence Thresholds** - Only auto-respond when AI is >90% confident
- **Sentiment Analysis** - Detects angry/frustrated customers and escalates to humans
- **Human Approval Gates** - Customer Support responses require Slack approval before sending
- **Context Awareness** - Analyzes full email threads and sender history
- **Safety Layers** - Spam detection, rate limiting, PII detection
- **Audit Trail** - Complete logging to Supabase for compliance and learning
- **Interactive Notifications** - Slack integration with approve/reject buttons

## What's New in v2.0

v2.0 is a complete rewrite focused on **safety, accuracy, and intelligence**:

- ✅ Switched from GPT-4O to Claude 3.5 Sonnet for better sentiment analysis
- ✅ Added confidence scores to all AI decisions (must be >90% to auto-process)
- ✅ Sentiment analysis detects frustrated/angry customers and escalates
- ✅ Customer Support now requires human approval (was auto-reply in v1.0)
- ✅ Full email thread context + sender history lookup
- ✅ Spam and phishing detection before processing
- ✅ Rate limiting (max 3 replies per thread to prevent loops)
- ✅ PII detection for Finance emails
- ✅ Complete audit trail logged to Supabase
- ✅ Interactive Slack notifications replace passive Telegram alerts

[See full comparison →](docs/V1_VS_V2_COMPARISON.md)

## Quick Start

### Prerequisites

- n8n (v1.0+) - [Install Guide](https://docs.n8n.io/hosting/)
- Anthropic Claude API key - [Get here](https://console.anthropic.com/)
- Gmail account with API access
- Slack workspace (recommended) or Telegram
- Supabase account (free tier) - [Sign up](https://supabase.com/)

### Installation (5 minutes)

1. **Set up Supabase database**
   ```sql
   -- Run the SQL schema in docs/SETUP.md Step 1.2
   ```

2. **Import workflow to n8n**
   - Download `workflows/v2.0-inbox-management-FIXED.json` (uses direct HTTP
     Request nodes to call Claude — see WORKFLOW_FIX_NOTES.md for why this
     replaced the original LangChain sub-node version, which n8n can't run
     standalone)
   - Import into n8n
   - Configure credentials

3. **Update configuration**
   - Set Slack channel IDs
   - Update Gmail label IDs
   - Configure finance email address

4. **Test and activate**
   - Send test emails
   - Verify classifications
   - Activate workflow

[Full setup guide →](docs/SETUP.md)

## How It Works

```
┌─────────────┐
│ Gmail Inbox │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│ Spam Detection  │ ← Filters phishing/spam
└────────┬────────┘
         │
         ▼
┌────────────────────┐
│ Context Enrichment │ ← Gets thread history + sender info
└─────────┬──────────┘
          │
          ▼
┌──────────────────────┐
│ Sentiment Analysis   │ ← Detects angry/frustrated tone
│    (Claude 3.5)      │
└──────────┬───────────┘
           │
           ▼
┌─────────────────────┐
│  Classification     │ ← Categorizes with confidence score
│    (Claude 3.5)     │
└──────────┬──────────┘
           │
           ▼
    ┌──────────────┐
    │ Confidence   │
    │ Gate (>90%)  │
    └───┬──────┬───┘
        │      │
     YES│      │NO → Needs Review Queue
        │      │
        ▼      ▼
┌─────────────────────┐
│  Smart Routing      │
│  - High Priority    │ → Draft + Slack notification
│  - Customer Support │ → Slack approval required ⚠️
│  - Promotions       │ → Summary + recommendation
│  - Finance/Billing  │ → PII check + forward
└─────────┬───────────┘
          │
          ▼
┌──────────────────┐
│ Audit Trail      │ ← Logs to Supabase
│  (Supabase)      │
└──────────────────┘
```

## Email Categories

### 🔴 High Priority
**Triggers:** Urgent keywords, key stakeholder emails, time-sensitive requests

**Action:**
- Creates Gmail draft
- Sends Slack notification with preview
- Shows AI confidence score
- Human manually sends after review

### 🎫 Customer Support
**Triggers:** Support requests, customer inquiries, feedback

**Action:**
- ✅ **Sentiment check first** - escalates if frustrated/angry
- Claude drafts 2 response options (primary + alternative)
- **Requires Slack approval** via interactive buttons
- Only sends after human clicks "Approve & Send"

**Safety:** This is the key improvement over v1.0 - no more risky auto-replies!

### 🎁 Promotions
**Triggers:** Marketing emails, newsletters, promotional offers

**Action:**
- Claude analyzes value (score 1-10)
- Provides summary + recommendation
- Extracts key points
- Sends to Slack for review
- No action required

### 💰 Finance/Billing
**Triggers:** Invoices, payment reminders, billing inquiries

**Action:**
- **PII detection** - flags SSN, credit cards, etc.
- Claude summarizes with amounts/dates
- Auto-forwards to finance team
- Slack notification with PII warning if detected

## Configuration

### Adjusting Confidence Thresholds

Current default: **90%** (recommended for balanced safety/automation)

To change:
1. Open workflow in n8n
2. Find node: "Confidence Gate (>90% + No Escalation)"
3. Update threshold value

**Guidelines:**
- 95% = Very conservative (more human review)
- 90% = Balanced (recommended) ⭐
- 85% = Aggressive (watch carefully)
- 80% = Very aggressive (not recommended)

### Rate Limiting

Current limits:
- Max **3 auto-replies per thread** (prevents loops)
- Max **10 auto-replies per sender per day**

Update in "Rate Limit Check" function node.

### AI Model Selection

**Current:** Claude 3.5 Sonnet (`claude-3-5-sonnet-20241022`)

**Why Claude over GPT-4O?**
- Better sentiment analysis (critical for customer support)
- More conservative responses (safer for auto-replies)
- Superior at detecting subtle frustration/anger
- Better instruction following for complex workflows

**Alternative:** Keep using GPT-4O if you prefer
- Update all Claude nodes to OpenAI nodes
- Adjust prompts for GPT-4O format
- Cost: ~3x cheaper but less nuanced

## Project Structure

```
inbox-management-improved/
├── workflows/
│   ├── v2.0-inbox-management-FIXED.json     ← Main workflow (use this one)
│   ├── v2.0-inbox-management-improved.json  ← Pre-fix version, kept for reference
│   └── archive/
│       ├── v1.0-original.json               ← Original for reference
│       └── v1.0-workflow-diagram.png
├── WORKFLOW_FIX_NOTES.md                    ← LangChain -> HTTP node fix writeup
├── docs/
│   ├── SETUP.md                             ← Complete setup guide
│   ├── V1_VS_V2_COMPARISON.md               ← Feature comparison
│   └── [future: API.md, TROUBLESHOOTING.md]
├── tests/
│   └── [future: sample test emails]
├── PROJECT_NOTES.md                         ← Design decisions
└── README.md                                ← You are here
```

## Monitoring & Analytics

### Supabase Audit Trail

Every email is logged with:
- Classification decision
- Confidence score
- Sentiment analysis results
- Action taken
- Timestamp

**Example queries:**

```sql
-- Classification accuracy by category
SELECT
  category,
  AVG(confidence_score) as avg_confidence,
  COUNT(*) as total_emails
FROM email_audit_log
WHERE processed_at >= NOW() - INTERVAL '7 days'
GROUP BY category;

-- Daily escalation rate
SELECT
  DATE(processed_at) as date,
  COUNT(*) as total,
  SUM(escalation_needed::int) as escalations,
  ROUND(100.0 * SUM(escalation_needed::int) / COUNT(*), 2) as escalation_rate
FROM email_audit_log
GROUP BY DATE(processed_at)
ORDER BY date DESC;

-- VIP sender activity
SELECT
  email,
  total_emails,
  last_contact_at,
  is_vip
FROM sender_history
WHERE is_vip = true
ORDER BY last_contact_at DESC;
```

### Slack Notifications

All notifications include:
- Email preview
- AI confidence score
- Sentiment analysis (if applicable)
- Detected issues (frustrated_language, legal_threat, etc.)
- Direct link to Gmail

## Security & Compliance

### Safety Features

1. **Spam/Phishing Detection** - Quarantines suspicious emails before processing
2. **Rate Limiting** - Prevents auto-reply loops (max 3 per thread)
3. **PII Detection** - Flags sensitive data in Finance emails
4. **Sentiment Escalation** - Routes angry customers to humans
5. **Confidence Thresholds** - Only auto-processes high-confidence decisions
6. **Audit Trail** - Complete logging for compliance

### Best Practices

- ✅ Use service_role key for Supabase (not anon key)
- ✅ Rotate API keys quarterly
- ✅ Never commit credentials to version control
- ✅ Review "Needs Review" queue daily
- ✅ Monitor confidence scores weekly
- ✅ Back up Supabase audit logs monthly

## Cost Estimate

**For 100 emails/day:**

- Claude API: ~$30/month
- Supabase: $0 (free tier sufficient)
- Slack: $0 (free tier sufficient)
- Gmail API: $0 (within quotas)

**Total: ~$30/month**

Scales linearly with email volume. [See detailed breakdown →](docs/SETUP.md#cost-estimates)

## Troubleshooting

### Common Issues

**Claude API errors**
- Verify API key is valid and has credits
- Check rate limits (Claude has per-minute limits)

**Low confidence scores**
- Review AI prompts for clarity
- Check if emails are genuinely ambiguous
- Consider adding examples to prompts

**Gmail trigger not firing**
- Verify OAuth token is still valid
- Check Gmail API quotas
- Ensure labels exist in Gmail

[Full troubleshooting guide →](docs/SETUP.md#troubleshooting)

## Roadmap

### v2.1 (Planned)
- [ ] Slack interactive button handlers (currently manual)
- [ ] Redis integration for better rate limiting
- [ ] Automatic feedback loop (learn from corrections)
- [ ] Additional language support

### v2.2 (Future)
- [ ] Custom classification categories
- [ ] Multi-inbox support
- [ ] Advanced analytics dashboard
- [ ] A/B testing for response templates

## Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Test thoroughly with sample emails
4. Submit pull request with description

## Support

- **Issues:** Create a GitHub issue
- **Questions:** Check [SETUP.md](docs/SETUP.md) and [Comparison guide](docs/V1_VS_V2_COMPARISON.md)
- **Feature requests:** Open a GitHub issue with use case

## License

MIT — see [LICENSE](LICENSE).

## Credits

- Built on [n8n](https://n8n.io/) workflow automation platform
- Powered by [Anthropic Claude](https://www.anthropic.com/)
- Original v1.0 concept by [Original Author]

## Changelog

### v2.0 (2025-10-13)
- Complete rewrite with Claude integration
- Added confidence thresholds and sentiment analysis
- Implemented human approval gates for Customer Support
- Added context enrichment (thread history + sender lookup)
- Implemented safety layers (spam, rate limiting, PII detection)
- Created comprehensive audit trail with Supabase
- Migrated from Telegram to Slack with interactive buttons

### v1.0 (2025-02-21)
- Initial release with GPT-4O
- Basic email classification (4 categories)
- Simple Gmail actions
- Telegram notifications

---

**Status:** Production-ready v2.0
**Minimum n8n Version:** 1.0+
**Last Updated:** 2025-10-13

⭐ If this workflow helps your business, please star the repository!
