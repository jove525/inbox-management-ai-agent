# Inbox Management AI Agent v2.0 - Setup Guide

## Overview

This guide will help you set up and configure the enhanced v2.0 Inbox Management AI Agent with Claude integration, confidence thresholds, sentiment analysis, and human-in-the-loop approvals.

---

## Prerequisites

### Required Services

1. **n8n** (v1.0+)
   - Self-hosted or n8n Cloud
   - [Installation Guide](https://docs.n8n.io/hosting/)

2. **Anthropic Claude API**
   - Sign up at [Anthropic Console](https://console.anthropic.com/)
   - Get API key with access to Claude 3.5 Sonnet or Claude 4
   - Recommended: Claude 3.5 Sonnet (best balance of speed/quality)

3. **Gmail Account**
   - Gmail account with API access enabled
   - OAuth2 credentials configured in n8n

4. **Slack Workspace** (Recommended)
   - Slack workspace for notifications
   - Slack app with permissions for posting messages and interactive buttons
   - Alternative: Keep using Telegram (less features)

5. **Supabase Account** (Required for audit trail)
   - Free tier is sufficient for most use cases
   - Sign up at [supabase.com](https://supabase.com/)

---

## Installation Steps

### Step 1: Set Up Supabase Database

#### 1.1 Create Project
1. Log in to Supabase
2. Create a new project
3. Note your project URL and API key (anon/public key)

#### 1.2 Create Database Tables

Run the following SQL in Supabase SQL Editor:

```sql
-- Sender History Table
CREATE TABLE sender_history (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email TEXT NOT NULL UNIQUE,
  first_contact_at TIMESTAMP DEFAULT NOW(),
  last_contact_at TIMESTAMP DEFAULT NOW(),
  total_emails INTEGER DEFAULT 1,
  is_vip BOOLEAN DEFAULT FALSE,
  notes TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Email Audit Log Table
CREATE TABLE email_audit_log (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email_id TEXT NOT NULL,
  thread_id TEXT,
  from_email TEXT NOT NULL,
  subject TEXT,
  category TEXT,
  confidence_score NUMERIC,
  sentiment TEXT,
  escalation_needed BOOLEAN DEFAULT FALSE,
  action_taken TEXT,
  processed_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Rate Limiting Tracker Table
CREATE TABLE rate_limit_tracker (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  thread_id TEXT NOT NULL,
  sender_email TEXT NOT NULL,
  reply_count INTEGER DEFAULT 1,
  last_reply_at TIMESTAMP DEFAULT NOW(),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Create indexes for performance
CREATE INDEX idx_sender_history_email ON sender_history(email);
CREATE INDEX idx_audit_log_email_id ON email_audit_log(email_id);
CREATE INDEX idx_audit_log_processed_at ON email_audit_log(processed_at);
CREATE INDEX idx_rate_limit_thread ON rate_limit_tracker(thread_id);
CREATE INDEX idx_rate_limit_sender ON rate_limit_tracker(sender_email);
```

#### 1.3 Set Row Level Security (Optional but Recommended)

```sql
-- Enable RLS
ALTER TABLE sender_history ENABLE ROW LEVEL SECURITY;
ALTER TABLE email_audit_log ENABLE ROW LEVEL SECURITY;
ALTER TABLE rate_limit_tracker ENABLE ROW LEVEL SECURITY;

-- Create policies (adjust based on your auth setup)
CREATE POLICY "Allow service account full access to sender_history"
  ON sender_history FOR ALL
  USING (auth.role() = 'service_role');

CREATE POLICY "Allow service account full access to email_audit_log"
  ON email_audit_log FOR ALL
  USING (auth.role() = 'service_role');

CREATE POLICY "Allow service account full access to rate_limit_tracker"
  ON rate_limit_tracker FOR ALL
  USING (auth.role() = 'service_role');
```

---

### Step 2: Configure Gmail API

#### 2.1 Create Gmail OAuth Credentials

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing
3. Enable Gmail API
4. Create OAuth 2.0 credentials:
   - Application type: Web application
   - Authorized redirect URIs: Your n8n OAuth callback URL
     - Format: `https://your-n8n-instance.com/rest/oauth2-credential/callback`
5. Download credentials JSON
6. Note: Client ID and Client Secret

#### 2.2 Configure in n8n

1. Go to n8n → Credentials
2. Add new credential → Gmail OAuth2 API
3. Enter Client ID and Client Secret
4. Authenticate with your Gmail account
5. Grant required permissions

#### 2.3 Create Gmail Labels

Create the following labels in Gmail (or update label IDs in workflow):

- `NEEDS_REVIEW_HIGH_PRIORITY`
- `CUSTOMER_SUPPORT_PENDING_APPROVAL`
- `PROMOTIONS`
- `FINANCE_BILLING`
- `NEEDS_REVIEW_LOW_CONFIDENCE`
- `SPAM_QUARANTINE`

To get label IDs, use Gmail API Explorer or run this in n8n:
```javascript
// Use Gmail node with operation: "Get All Labels"
```

---

### Step 3: Configure Anthropic Claude

#### 3.1 Get API Key

1. Sign up at [Anthropic Console](https://console.anthropic.com/)
2. Navigate to API Keys
3. Create a new API key
4. Copy the key (starts with `sk-ant-...`)

#### 3.2 Configure in n8n

1. Go to n8n → Credentials
2. Add new credential → Anthropic API
3. Enter your API key
4. Test connection

#### 3.3 Model Selection

The workflow uses **Claude 3.5 Sonnet** (`claude-3-5-sonnet-20241022`) by default.

**Alternative models:**
- `claude-3-opus-20240229` - Most capable, slower, more expensive
- `claude-3-sonnet-20240229` - Previous generation
- `claude-3-haiku-20240307` - Fastest, cheapest, less nuanced

**Cost Comparison (per 1M tokens):**
- Claude 3.5 Sonnet: $3 input / $15 output
- Claude 3 Opus: $15 input / $75 output
- Claude 3 Haiku: $0.25 input / $1.25 output

For this workflow, **Claude 3.5 Sonnet** is recommended for the best balance.

---

### Step 4: Configure Slack (Recommended)

#### 4.1 Create Slack App

1. Go to [Slack API](https://api.slack.com/apps)
2. Click "Create New App" → From scratch
3. Name: "Inbox Management Agent"
4. Select your workspace

#### 4.2 Configure Permissions

Add these OAuth scopes:
- `chat:write` - Post messages
- `chat:write.public` - Post to public channels
- `files:write` - Upload files (optional)
- `incoming-webhook` - Post notifications

For interactive buttons (Customer Support approval):
- `commands` - Slash commands
- `im:write` - Direct messages
- `channels:read` - Read channel info

#### 4.3 Install App to Workspace

1. Install app to your workspace
2. Copy the Bot User OAuth Token (starts with `xoxb-`)
3. Get your channel ID:
   - Right-click channel → View channel details → Copy channel ID
   - Format: `C01234567`

#### 4.4 Configure in n8n

1. Go to n8n → Credentials
2. Add new credential → Slack OAuth2 API
3. Enter Bot User OAuth Token
4. Test connection

#### 4.5 Update Workflow Channel IDs

In the workflow JSON, replace `YOUR_SLACK_CHANNEL_ID` with your actual channel ID in these nodes:
- `Notify Slack: High Priority`
- `Slack Approval: Customer Support`
- `Notify Slack: Promotions`
- `Notify Slack: Finance`
- `Notify Slack: Needs Review`
- `Notify Slack: Spam Detected`

---

### Step 5: Configure Supabase in n8n

#### 5.1 Add Supabase Credentials

1. Go to n8n → Credentials
2. Add new credential → Supabase
3. Enter:
   - **Host**: Your Supabase project URL (e.g., `https://abc123.supabase.co`)
   - **Service Role Key**: Your Supabase service_role key (NOT the anon key)
     - Find in: Supabase → Project Settings → API → service_role key

#### 5.2 Test Connection

1. Create a test workflow with Supabase node
2. Try a simple SELECT query on `sender_history`
3. Verify connection works

---

### Step 6: Import Workflow to n8n

#### 6.1 Import JSON

1. Open n8n
2. Click "Add workflow" → "Import from File"
3. Select `workflows/v2.0-inbox-management-improved.json`
4. Workflow imports with all nodes

#### 6.2 Update Credentials

Go through each node and assign credentials:

**Gmail nodes:**
- Gmail Trigger
- Get Email Thread History
- Get Thread Messages
- All label nodes
- Create Draft
- Forward to Finance Dept

**Claude nodes:**
- Sentiment Analysis (Claude)
- Classification with Confidence (Claude)
- Draft High Priority Response (Claude)
- Draft Support Response (Claude)
- Analyze Promotion (Claude)
- Summarize for Finance (Claude)

**Slack nodes:**
- All "Notify Slack" nodes

**Supabase nodes:**
- Lookup Sender History
- Log to Audit Trail (Supabase)

#### 6.3 Update Configuration Values

**Finance Email:**
- Node: "Forward to Finance Dept"
- Field: `sendTo`
- Update: `finance@abccorp.com` → Your finance team email

**Customer Support Email:**
- Node: "Draft Support Response (Claude)"
- Update the referral email: `customersupport@abccorp.com` → Your support email

**Gmail Label IDs:**
- Get label IDs from Gmail (Step 2.3)
- Update in all label nodes

**Slack Channel IDs:**
- Update `YOUR_SLACK_CHANNEL_ID` in all Slack nodes (Step 4.5)

---

### Step 7: Test the Workflow

#### 7.1 Send Test Emails

Create test emails for each category:

**High Priority Test:**
```
Subject: URGENT: Project deadline moved to today
From: boss@company.com
Body: Hi, the client just called and needs the presentation by EOD. Can you prioritize this?
```

**Customer Support Test:**
```
Subject: Help with login issues
From: customer@example.com
Body: Hi, I'm having trouble logging into my account. Can you help?
```

**Promotions Test:**
```
Subject: 50% off sale this weekend!
From: marketing@vendor.com
Body: Don't miss our biggest sale of the year...
```

**Finance Test:**
```
Subject: Invoice #12345 payment reminder
From: billing@vendor.com
Body: This is a reminder that invoice #12345 for $500 is due...
```

**Spam Test:**
```
Subject: URGENT: Verify your account immediately
From: suspicious@temp-mail.org
Body: Click here immediately to verify your account or it will be suspended...
```

#### 7.2 Verify Workflow Execution

1. Manually trigger workflow or wait for email poll
2. Check n8n execution log
3. Verify:
   - Emails are classified correctly
   - Confidence scores are reasonable
   - Sentiment analysis is accurate
   - Slack notifications are sent
   - Supabase logs are created
   - Gmail labels are applied

#### 7.3 Test Interactive Approvals

1. Send a Customer Support test email
2. Check Slack for approval message
3. Click "Approve & Send" button
4. Verify response is sent (Note: You'll need to set up Slack interactivity webhook - see Advanced Configuration)

---

### Step 8: Activate Workflow

1. Once testing is complete, click "Active" toggle in n8n
2. Workflow now runs automatically every minute
3. Monitor executions for first few days

---

## Configuration Options

### Adjusting Confidence Thresholds

**Current Setting:** 90% confidence required for automation

**To change:**
1. Open workflow
2. Find node: "Confidence Gate (>90% + No Escalation)"
3. Update condition: `value2` from 90 to your desired threshold

**Recommendations:**
- Conservative (95%): Very safe, more human review needed
- Balanced (90%): Recommended starting point
- Aggressive (85%): More automation, slightly more risk
- Very Aggressive (80%): High automation, watch carefully

### Adjusting Rate Limits

**Current Settings:**
- Max 3 auto-replies per thread
- Max 10 auto-replies per sender

**To change:**
1. Open workflow
2. Find node: "Rate Limit Check"
3. Update variables:
   - `maxAutoRepliesPerThread`
   - `maxAutoRepliesPerSender`

### Customizing AI Prompts

All Claude nodes have customizable prompts. To adjust:

1. Open the specific Claude node
2. Edit the prompt in the "messages" field
3. Keep the JSON output format requirements
4. Test thoroughly after changes

**Key nodes to customize:**
- **Sentiment Analysis** - Adjust what emotional triggers to detect
- **Classification** - Modify category definitions
- **Response Generation** - Change tone and style

### Spam Detection Patterns

**To add more spam keywords:**
1. Open "Spam/Phishing Detection" function node
2. Add to `spamKeywords` array
3. Add regex patterns to `suspiciousPatterns`

---

## Advanced Configuration

### Setting Up Slack Interactive Buttons

For Customer Support approval buttons to work:

1. In Slack App settings → Interactivity & Shortcuts
2. Enable Interactivity
3. Request URL: Your n8n webhook URL
   - Create a new n8n webhook workflow to handle button clicks
   - URL format: `https://your-n8n-instance.com/webhook/slack-approval`

4. Create handler workflow in n8n:
   - Webhook trigger
   - Process button action
   - Send email reply if approved
   - Update Supabase logs

### Implementing Rate Limiting with Redis

For production-grade rate limiting:

1. Set up Redis instance
2. Replace Function node logic with Redis queries
3. Track:
   - Thread reply counts
   - Sender reply counts
   - Last reply timestamps

### Adding PII Redaction

Currently PII is only detected. To redact:

1. Update "PII Detection" function node
2. Add redaction logic:
```javascript
// Replace detected PII with [REDACTED]
text = text.replace(patterns.ssn, '[SSN REDACTED]');
text = text.replace(patterns.credit_card, '[CARD REDACTED]');
```

### Creating Dashboard for Analytics

Use Supabase + any BI tool:

**Metrics to track:**
- Classification accuracy (feedback from users)
- Average confidence scores by category
- Escalation rate
- Response time
- Auto-reply success rate

**Example Query:**
```sql
SELECT
  category,
  AVG(confidence_score) as avg_confidence,
  COUNT(*) as total_emails,
  SUM(CASE WHEN escalation_needed THEN 1 ELSE 0 END) as escalations
FROM email_audit_log
WHERE processed_at >= NOW() - INTERVAL '7 days'
GROUP BY category;
```

---

## Monitoring & Maintenance

### Daily Checks

- Review "Needs Review" emails in Slack
- Check Supabase audit logs for anomalies
- Verify no emails stuck in spam quarantine

### Weekly Reviews

- Analyze classification accuracy
- Review false positives/negatives
- Adjust confidence thresholds if needed
- Check rate limit hits

### Monthly Tasks

- Review sender history for VIP flagging
- Analyze sentiment detection accuracy
- Update spam detection patterns
- Review and optimize costs

---

## Troubleshooting

### Common Issues

**Issue: Claude API errors**
- Check API key is valid
- Verify you have sufficient credits
- Check rate limits (Claude has request limits)

**Issue: Gmail trigger not firing**
- Verify OAuth token is still valid
- Check Gmail API quotas
- Ensure labels exist

**Issue: Supabase connection failed**
- Verify service_role key (not anon key)
- Check RLS policies don't block service account
- Verify table names match exactly

**Issue: Slack notifications not sending**
- Verify bot token has correct scopes
- Check channel ID is correct
- Ensure bot is invited to channel

**Issue: Low confidence scores**
- Emails might genuinely be ambiguous
- Review AI prompts for clarity
- Consider training data / examples in prompts

### Debug Mode

Enable debug logging in n8n:
1. Settings → Environments
2. Add: `N8N_LOG_LEVEL=debug`
3. Restart n8n
4. Check logs for detailed execution info

---

## Security Best Practices

1. **API Keys**: Never commit API keys to version control
2. **Gmail OAuth**: Rotate tokens periodically
3. **Supabase**: Use service_role key only in n8n, never expose to frontend
4. **Rate Limiting**: Implement strict limits to prevent abuse
5. **PII Handling**: Enable PII redaction for sensitive industries
6. **Audit Trail**: Regularly backup Supabase data
7. **Access Control**: Restrict who can modify n8n workflows

---

## Cost Estimates

### Expected Costs (100 emails/day)

**Claude API:**
- ~300 emails/day with sentiment + classification + drafting
- Average: 2K tokens input + 500 tokens output per email
- Cost: ~$0.50 - $1.00/day = $15-30/month

**Supabase:**
- Free tier sufficient for <500K rows
- $0/month

**Slack:**
- Free plan sufficient
- $0/month

**Gmail API:**
- Free (within quotas)
- $0/month

**Total: ~$15-30/month for 100 emails/day**

Scales linearly with email volume.

---

## Migration from v1.0

If you're upgrading from the original workflow:

1. **Backup v1.0**: Already done in `workflows/archive/`
2. **Test v2.0 in parallel**: Run both for 1 week
3. **Compare results**: Check accuracy improvements
4. **Gradually transition**: Move categories one at a time
5. **Deactivate v1.0**: Once confident in v2.0

---

## Support & Contributing

**Issues & Questions:**
- Create an issue in the project repository
- Check PROJECT_NOTES.md for design decisions

**Feature Requests:**
- Submit via GitHub issues
- Include use case and expected behavior

**Contributing:**
- Fork repository
- Create feature branch
- Submit pull request with tests

---

## Next Steps

After basic setup is complete:

1. ✅ Run workflow for 1 week in monitoring mode
2. ✅ Analyze audit logs for accuracy
3. ✅ Fine-tune confidence thresholds
4. ✅ Add more spam patterns based on your inbox
5. ✅ Create VIP sender list in Supabase
6. ✅ Set up analytics dashboard
7. ✅ Implement feedback mechanism for continuous learning

---

**Version:** 2.0
**Last Updated:** 2025-10-13
**Minimum n8n Version:** 1.0+
**Recommended n8n Version:** 1.20+
