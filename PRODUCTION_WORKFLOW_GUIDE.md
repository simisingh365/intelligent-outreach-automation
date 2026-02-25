# Job Application Agent - Production Workflow Guide

## 🎯 Overview

This is a complete, production-grade n8n workflow that automates job applications by:
- Scraping Indeed job postings via RSS
- Analyzing job fit using Chutes AI and your resume
- Generating personalized outreach emails
- Extracting contact emails from job descriptions
- Automatically sending emails or logging drafts
- Tracking everything in Google Sheets with deduplication

## 🏗️ Architecture

### Workflow Phases

```
Phase 1: Initialization
├── Manual Trigger (for testing)
└── Schedule Trigger (every 6 hours)
    ↓
Phase 2: Configuration
└── Set User Profile (resume, role, name, email)
    ↓
Phase 3: Data Ingestion
├── Fetch Indeed Jobs (RSS-to-JSON API)
├── Read Existing Jobs from Sheet
└── Deduplicate Jobs (Code Node)
    ↓
Phase 4: AI Processing
├── Chutes AI - Analyze Job Fit
└── Parse AI Response (Code Node)
    ↓
Phase 5: Email Extraction
└── Extract Contact Email (Regex)
    ↓
Phase 6: Decision & Action
└── IF: Email Found & Good Fit?
    ├── TRUE → Send Email → Log as SENT
    └── FALSE → Log as DRAFT/LOW SCORE
```

## 🔧 Setup Instructions

### 1. Environment Variables

Set these in your n8n instance (Settings → Variables):

```bash
USER_RESUME_TEXT="Your full resume text here..."
TARGET_JOB_ROLE="AI Automation Engineer"
USER_NAME="Your Name"
USER_EMAIL="your.email@example.com"
CHUTES_MODEL="Qwen/Qwen2.5-32B"
GOOGLE_SHEET_ID="your-google-sheet-id"
GOOGLE_SHEET_NAME="Sheet1"
```

### 2. Credentials Setup

#### A. Chutes AI Authentication
1. Go to Credentials → Add Credential
2. Select "HTTP Header Auth"
3. Name: `Chutes AI Auth`
4. Header Name: `Authorization`
5. Header Value: `Bearer YOUR_CHUTES_API_KEY`

#### B. Google Sheets OAuth2
1. Go to Credentials → Add Credential
2. Select "Google Sheets OAuth2 API"
3. Follow the OAuth flow to authorize

#### C. Gmail SMTP
1. Go to Credentials → Add Credential
2. Select "SMTP"
3. Configure:
   - Host: `smtp.gmail.com`
   - Port: `587`
   - Security: `STARTTLS`
   - User: Your Gmail address
   - Password: App-specific password (not your regular password)

**Note:** For Gmail, you need to:
- Enable 2FA on your Google account
- Generate an App Password at https://myaccount.google.com/apppasswords

### 3. Google Sheet Setup

Create a Google Sheet with these columns (Row 1):

| Timestamp | Job Title | Job Link | Company | AI Score | Status | Contact Email | Email Subject | Email Body | AI Reasoning |
|-----------|-----------|----------|---------|----------|--------|---------------|---------------|------------|--------------|

**Important:** The workflow reads Column C (Job Link) for deduplication.

## 🚀 How It Works

### 1. Dual Trigger System (THE FIX)
Both triggers connect to the same flow:
- **Manual Trigger**: For testing and immediate runs
- **Schedule Trigger**: Runs automatically every 6 hours

### 2. User Profile Configuration
Uses environment variables with fallback defaults. All job items are merged with this profile data so the AI has full context.

### 3. Indeed Job Fetching
Uses the RSS-to-JSON bridge API (`https://api.rss2json.com/v1/api.json`) which is more stable than n8n's native RSS node.

Example URL:
```
https://api.rss2json.com/v1/api.json?rss_url=https://www.indeed.com/rss?q=AI+Automation+Engineer&l=Remote&sort=date
```

### 4. Deduplication Logic
The Code node:
1. Fetches existing job links from Google Sheets (Column C)
2. Filters out any jobs already in the sheet
3. Only processes NEW jobs
4. Returns empty array if no new jobs found

### 5. AI Analysis (Chutes AI)
Sends a structured prompt to Chutes AI:
- **System Prompt**: "You are an expert recruiter..."
- **User Prompt**: Includes job description + resume
- **Response Format**: Forces JSON output with score, subject, body, reasoning
- **Error Handling**: `onError: continueErrorOutput` prevents workflow crashes

### 6. JSON Parsing
The Parse AI Response node:
- Extracts content from Chutes AI response
- Removes markdown code blocks if present
- Safely parses JSON
- Falls back to default values if parsing fails
- Never crashes the workflow

### 7. Email Extraction (Regex)
Uses this regex pattern:
```regex
[a-zA-Z0-9._-]+@[a-zA-Z0-9._-]+\.[a-zA-Z0-9._-]+
```

Searches the job description for email addresses and extracts the first valid match.

### 8. Smart Decision Logic
The IF node checks TWO conditions (AND logic):
1. `has_contact_email === true`
2. `ai_score >= 70`

**Path A (TRUE)**: Email found AND good fit
- Sends email via SMTP
- Logs to Sheet with Status: "SENT"

**Path B (FALSE)**: No email OR low score
- Skips email sending
- Logs to Sheet with Status: "DRAFT READY (NO CONTACT EMAIL)" or "LOW SCORE (< 70)"

## 📊 Google Sheets Logging

Every job is logged with:
- **Timestamp**: ISO 8601 format
- **Job Title**: From Indeed RSS
- **Job Link**: For deduplication
- **Company**: From RSS author field
- **AI Score**: 0-100 fit score
- **Status**: SENT / DRAFT READY / LOW SCORE
- **Contact Email**: Extracted or "Not Found"
- **Email Subject**: Generated by AI
- **Email Body**: Generated by AI
- **AI Reasoning**: Why it's a good/bad fit

## 🔍 Key Features

### ✅ Production-Ready Elements

1. **Dual Trigger System**: Manual + Scheduled (THE FIX)
2. **Deduplication**: Never processes the same job twice
3. **Error Handling**: AI failures don't crash the workflow
4. **Safe JSON Parsing**: Handles malformed AI responses
5. **Regex Email Extraction**: Finds contact emails in descriptions
6. **Smart Decision Logic**: Only sends when appropriate
7. **Complete Logging**: Every job tracked in Google Sheets
8. **Environment Variables**: Easy configuration without editing workflow
9. **Credential Management**: Secure API key storage

### 🎯 AI Prompt Engineering

The system prompt ensures:
- JSON-only responses (no extra text)
- Structured output format
- Fit scoring (0-100)
- Personalized email generation
- Brief reasoning for decisions

### 🛡️ Safety Features

1. **No Crashes**: All error-prone nodes have fallbacks
2. **No Duplicates**: Sheet-based deduplication
3. **No Spam**: Only sends when score >= 70 AND email found
4. **Full Audit Trail**: Every decision logged
5. **Timeout Protection**: 30-second timeout on AI requests

## 🔄 Workflow Execution Flow

```
START (Manual or Schedule)
  ↓
Set User Profile (merge resume + config)
  ↓
Fetch Indeed Jobs (RSS-to-JSON)
  ↓
Read Existing Jobs from Sheet
  ↓
Deduplicate (filter out existing)
  ↓
[For each NEW job]
  ↓
Chutes AI Analysis (job + resume → score + email)
  ↓
Parse AI Response (safe JSON parsing)
  ↓
Extract Contact Email (regex search)
  ↓
IF (email found AND score >= 70)?
  ├─ YES → Send Email → Log as SENT
  └─ NO → Log as DRAFT/LOW SCORE
```

## 📝 Customization Options

### Change Job Search Query
Edit the "Fetch Indeed Jobs" node URL:
```javascript
https://api.rss2json.com/v1/api.json?rss_url=https://www.indeed.com/rss?q=YOUR+JOB+TITLE&l=YOUR+LOCATION&sort=date
```

### Change AI Model
Set environment variable:
```bash
CHUTES_MODEL="Qwen/Qwen2.5-32B"
# or
CHUTES_MODEL="meta-llama/Llama-3.3-70B-Instruct"
```

### Change Score Threshold
Edit the IF node condition:
```javascript
$json.ai_score >= 70  // Change 70 to your threshold
```

### Change Schedule Frequency
Edit the Schedule Trigger:
- Current: Every 6 hours
- Options: hourly, daily, weekly, custom cron

## 🐛 Troubleshooting

### No Jobs Being Processed
1. Check if Indeed RSS feed is returning results
2. Verify deduplication isn't filtering everything
3. Check Google Sheets connection

### AI Parsing Errors
- The workflow handles this gracefully
- Check "parsing_error" field in Sheet logs
- Verify Chutes AI API key is valid

### Emails Not Sending
1. Verify Gmail SMTP credentials
2. Check if App Password is correct
3. Ensure 2FA is enabled on Google account
4. Check spam folder for test emails

### Deduplication Not Working
1. Verify Google Sheet has "Job Link" column (Column C)
2. Check if Sheet ID is correct in env vars
3. Ensure OAuth2 credentials are authorized

## 🚀 Deployment Checklist

- [ ] Set all environment variables
- [ ] Configure Chutes AI credentials
- [ ] Configure Google Sheets OAuth2
- [ ] Configure Gmail SMTP
- [ ] Create Google Sheet with correct columns
- [ ] Test with Manual Trigger first
- [ ] Verify email sending works
- [ ] Verify Sheet logging works
- [ ] Enable Schedule Trigger
- [ ] Monitor first few automated runs

## 📚 LinkedIn Integration (Future)

Currently stubbed out because:
- LinkedIn blocks direct scraping
- Requires paid SerpApi integration
- Would need API key + subscription

**Future Implementation:**
```javascript
// Use SerpApi LinkedIn Jobs API
https://serpapi.com/search?engine=linkedin_jobs&q=AI+Engineer&location=Remote
```

## 🎓 Best Practices

1. **Start with Manual Trigger**: Test thoroughly before enabling schedule
2. **Monitor First Week**: Check Sheet logs daily
3. **Adjust Score Threshold**: Based on email quality
4. **Update Resume Regularly**: Keep USER_RESUME_TEXT current
5. **Review Drafts**: Check low-score jobs manually
6. **Backup Sheet**: Export regularly
7. **Rate Limiting**: 6-hour schedule prevents API abuse

## 📊 Expected Performance

- **Jobs per Run**: 5-20 (depends on Indeed results)
- **Processing Time**: ~5-10 seconds per job
- **AI Response Time**: 2-5 seconds
- **Email Send Time**: 1-2 seconds
- **Total Run Time**: 1-3 minutes for 10 jobs

## 🔐 Security Notes

- Never commit API keys to version control
- Use environment variables for all secrets
- Rotate API keys periodically
- Monitor Sheet for unauthorized access
- Use App Passwords for Gmail (not main password)

## 📞 Support

For issues with:
- **n8n**: Check n8n community forum
- **Chutes AI**: Contact Chutes AI support
- **Google Sheets**: Verify OAuth2 scopes
- **Gmail SMTP**: Check Google account security settings

---

**Version**: 1.0.0  
**Last Updated**: 2026-02-25  
**Compatibility**: n8n v1.0+, Chutes AI Pro Plan
