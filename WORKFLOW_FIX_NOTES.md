# Workflow Fix Notes - LangChain to HTTP Request Conversion

**Date:** October 16, 2025
**Issue:** LangChain Claude nodes cannot be used as standalone nodes in n8n
**Solution:** Replaced all 6 LangChain nodes with HTTP Request nodes that call the Anthropic API directly

---

## What Was Changed

### Problem Identified

The original workflow (`v2.0-inbox-management-improved.json`) used **LangChain Chat Anthropic** nodes:
```json
"type": "@n8n/n8n-nodes-langchain.lmChatAnthropic"
```

These are **sub-nodes** designed to work inside AI Agent/Chain parent nodes, not as standalone nodes. They wouldn't execute properly when connected directly in the workflow.

### Solution Applied

Created a fixed workflow (`v2.0-inbox-management-FIXED.json`) that replaces all 6 LangChain nodes with **HTTP Request** nodes that call the Anthropic API directly.

---

## Nodes Replaced (6 Total)

### 1. Sentiment Analysis (Claude)
- **Old:** LangChain Chat Anthropic node
- **New:** HTTP Request node
- **Endpoint:** POST `https://api.anthropic.com/v1/messages`
- **Purpose:** Analyzes email sentiment, urgency, and escalation needs
- **Output:** Returns JSON with sentiment, urgency_level, escalation_needed, etc.

### 2. Classification with Confidence (Claude)
- **Old:** LangChain Chat Anthropic node
- **New:** HTTP Request node
- **Endpoint:** POST `https://api.anthropic.com/v1/messages`
- **Purpose:** Classifies email into categories (High Priority, Customer Support, Promotions, Finance)
- **Output:** Returns JSON with category, confidence_score, reasoning

### 3. Draft High Priority Response (Claude)
- **Old:** LangChain Chat Anthropic node
- **New:** HTTP Request node
- **Endpoint:** POST `https://api.anthropic.com/v1/messages`
- **Purpose:** Drafts professional responses for high-priority emails
- **Output:** Returns JSON with subject, message, tone

### 4. Draft Support Response (Claude)
- **Old:** LangChain Chat Anthropic node
- **New:** HTTP Request node
- **Endpoint:** POST `https://api.anthropic.com/v1/messages`
- **Purpose:** Drafts empathetic customer support responses
- **Output:** Returns JSON with message, alternative_message, escalation_recommendation

### 5. Analyze Promotion (Claude)
- **Old:** LangChain Chat Anthropic node
- **New:** HTTP Request node
- **Endpoint:** POST `https://api.anthropic.com/v1/messages`
- **Purpose:** Summarizes and evaluates promotional emails
- **Output:** Returns JSON with summary, recommendation, value_score, key_points

### 6. Summarize for Finance (Claude)
- **Old:** LangChain Chat Anthropic node
- **New:** HTTP Request node
- **Endpoint:** POST `https://api.anthropic.com/v1/messages`
- **Purpose:** Summarizes finance/billing emails for finance team
- **Output:** Returns JSON with subject, message, priority, action_required

---

## Key Configuration Changes

### HTTP Request Node Configuration

Each Claude node now uses:

**Method:** POST
**URL:** `https://api.anthropic.com/v1/messages`
**Authentication:** Predefined Credential Type → `anthropicApi`
**Headers:**
- `anthropic-version: 2023-06-01`

**Body Parameters:**
- `model`: `claude-3-5-sonnet-20241022`
- `max_tokens`: `1024` or `2048` (depending on use case)
- `temperature`: `0.2` to `0.7` (depending on task)
- `messages`: Array with single user message containing the prompt

### Response Format Change

**Old LangChain output:**
```javascript
$json.message.content.category
```

**New HTTP Request output:**
```javascript
JSON.parse($json.content[0].text).category
```

The Anthropic API returns responses in this format:
```json
{
  "content": [
    {
      "type": "text",
      "text": "{\"category\": \"High Priority\", \"confidence_score\": 95}"
    }
  ]
}
```

So all downstream nodes that reference Claude output now use `JSON.parse($json.content[0].text)` to extract the JSON response.

---

## Updated Expressions Throughout Workflow

### Confidence Gate Node
**Before:**
```javascript
{{ $json.message.content.confidence_score }}
```

**After:**
```javascript
{{ JSON.parse($json.content[0].text).confidence_score }}
```

### All Route Nodes (High Priority, Customer Support, Promotions, Finance)
**Before:**
```javascript
{{ $json.message.content.category }}
```

**After:**
```javascript
{{ JSON.parse($('Classification with Confidence (Claude)').item.json.content[0].text).category }}
```

### All Slack Notification Nodes
**Before:**
```javascript
{{ $('Sentiment Analysis (Claude)').item.json.message.content.sentiment }}
```

**After:**
```javascript
{{ JSON.parse($('Sentiment Analysis (Claude)').item.json.content[0].text).sentiment }}
```

### Create Draft Node
**Before:**
```javascript
{{ $json.message.content.subject }}
{{ $json.message.content.message }}
```

**After:**
```javascript
{{ JSON.parse($json.content[0].text).subject }}
{{ JSON.parse($json.content[0].text).message }}
```

### Supabase Audit Log Node
**Before:**
```javascript
{{ $('Classification with Confidence (Claude)').item.json.message.content.category }}
```

**After:**
```javascript
{{ JSON.parse($('Classification with Confidence (Claude)').item.json.content[0].text).category }}
```

---

## Additional Fixes Applied

### 1. Fixed Gmail Label Name
- **Changed:** `PROMOTIONS` → `MARKETING_PROMOTIONS`
- **Reason:** `PROMOTIONS` conflicts with Gmail system label
- **Location:** Label: Promotions node (line 462)

### 2. Fixed Message ID References
- **Updated:** Several Gmail nodes to reference `$('Gmail Trigger').item.json.id`
- **Reason:** Ensures correct message ID is used throughout workflow
- **Affected nodes:** Label nodes, rate limit checks

### 3. Improved PII Detection Node
- **Added:** Fallback to Gmail Trigger data if current node data is missing
- **Reason:** Prevents errors if data structure changes during flow

---

## Testing Checklist

Before activating the fixed workflow:

- [ ] Import `v2.0-inbox-management-FIXED.json` to n8n
- [ ] Assign Anthropic API credentials to all 6 HTTP Request nodes
- [ ] Assign Gmail credentials to all Gmail nodes
- [ ] Assign Supabase credentials to both Supabase nodes
- [ ] Test each Claude node manually (using "Test step" in n8n)
- [ ] Verify JSON parsing works correctly
- [ ] Send test emails to verify end-to-end flow

---

## Credentials Required

The fixed workflow requires:

1. **Anthropic API** - Used by 6 HTTP Request nodes
   - Get API key from: https://console.anthropic.com/
   - Configure in n8n as: "Anthropic API" credential type

2. **Gmail OAuth2** - Used by ~11 Gmail nodes
   - Already configured in your n8n instance

3. **Supabase** - Used by 2 Supabase nodes
   - Already configured in your n8n instance

---

## File Comparison

| File | Type | Status |
|------|------|--------|
| `v2.0-inbox-management-improved.json` | Original | ❌ LangChain nodes won't work |
| `v2.0-inbox-management-FIXED.json` | Fixed | ✅ Ready to test |

**Recommendation:** Use the **FIXED** version for all testing and production use.

---

## Next Steps

1. **Import the fixed workflow** to n8n
2. **Create test Gmail account** (recommended: `yourname.n8ntest@gmail.com`)
3. **Apply credentials** to all nodes
4. **Send test emails** for each category
5. **Monitor executions** and verify Claude responses are parsed correctly
6. **Debug any issues** with JSON parsing or API calls

---

## Troubleshooting

### If Claude nodes fail with "Invalid JSON" errors:
- Check that `JSON.parse($json.content[0].text)` is used correctly
- Verify Claude is returning valid JSON (check execution logs)
- Ensure prompts specify "Only output valid JSON, no explanation"

### If you see "anthropic-version header missing" errors:
- Verify the header is set to `2023-06-01` in all HTTP Request nodes

### If authentication fails:
- Verify Anthropic API credential is configured in n8n
- Check API key has credits ($5 added in previous session)
- Test API key directly: https://console.anthropic.com/settings/keys

---

**Created:** October 16, 2025
**Author:** Claude Code
**Version:** Fixed v2.0
