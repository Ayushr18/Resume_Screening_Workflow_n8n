# Automated Resume Relevance Scoring Workflow
**Resume Screening System**

---

## 📋 Overview

An intelligent, fully automated workflow that evaluates candidate resumes against a Product Intern job description using AI-powered analysis. The system continuously monitors a Google Drive folder, processes new resumes, and maintains a live Google Sheet with relevance scores and detailed reasoning.

---

## 🎯 Objective

Automate the resume screening process by:
- Reading resumes from a designated Google Drive folder
- Analyzing each resume against job requirements using AI
- Extracting candidate information and calculating relevance scores
- Maintaining structured results in Google Sheets for easy review

---

## 🏗️ System Architecture

### Folder Structure
```
Resume_Screening_System/
├── Resumes/              → Input folder for candidate PDF resumes
├── Job_Description/      → Contains the Product Intern JD (PDF)
└── Evaluation_Results    → Output Google Sheet with analysis results
```

### Workflow Components

**1. Schedule Trigger**
- Runs automatically every 15 minutes
- Ensures continuous monitoring for new resumes
- Can be adjusted to any frequency (hourly, daily, etc.)

**2. Search & Discovery**
- Searches the `Resumes` folder for all PDF files
- Uses Google Drive API with folder filtering
- Returns list of resume files with metadata

**3. Loop Over Items**
- Processes resumes sequentially (one at a time)
- Prevents API rate limiting issues
- Batch size: 1 item per iteration

**4. Resume Download**
- Downloads each PDF file from Google Drive
- Retrieves binary data for text extraction

**5. PDF Text Extraction**
- Extracts plain text from PDF resumes
- Handles various PDF formats
- Prepares content for AI analysis

**6. Request Payload Builder (Code Node)**
- Constructs the AI API request dynamically
- Includes:
  - Job description requirements
  - Extracted resume text
  - Scoring criteria
  - JSON response format specification
- Escapes special characters for valid JSON

**7. AI Analysis (HTTP Request - Groq API)**
- Model: `llama-3.3-70b-versatile`
- Analyzes resume against job requirements
- Evaluates:
  - Skills match (40% weight)
  - Experience relevance (30% weight)
  - Education alignment (15% weight)
  - Additional qualifications (15% weight)
- Generates relevance score (0-100)
- Provides detailed reasoning

**8. Response Parser (Code Node)**
- Extracts JSON from AI response
- Handles edge cases:
  - Markdown code blocks
  - Control characters
  - Malformed JSON
- Combines with resume metadata
- Adds timestamp

**9. Google Sheets Integration**
- Operation: Append or Update Row
- Duplicate detection using File ID
- Columns saved:
  - Candidate Name
  - Mobile Number
  - Email Address
  - Resume Link (Google Drive)
  - Relevance Score (0-100)
  - Detailed Reasoning
  - Date Processed
  - File ID (for duplicate checking)

**10. Rate Limiter (Wait Node)**
- Waits 10 seconds between each resume
- Prevents API rate limit errors
- Groq free tier: 30 requests/minute
- Our rate: 6 requests/minute (safe buffer)

**11. Loop Back**
- Returns to process next resume
- Continues until all resumes processed

---

## 🔄 Complete Workflow Flow
```
┌─────────────────────────────────────────────────────────────────┐
│  Schedule Trigger (Every 15 minutes)                            │
└─────────────────────┬───────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────────┐
│  Search Files & Folders (Google Drive)                          │
│  └─ Folder: Resumes                                             │
│  └─ File type: .pdf                                             │
│  └─ Result: List of PDF resumes                                 │
└─────────────────────┬───────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────────┐
│  Loop Over Items                                                 │
│  └─ Batch size: 1 (process one resume at a time)               │
└─────────────────────┬───────────────────────────────────────────┘
                      ↓ (for each resume)
┌─────────────────────────────────────────────────────────────────┐
│  Download File (Google Drive)                                    │
│  └─ Downloads PDF binary data                                   │
└─────────────────────┬───────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────────┐
│  Extract from File (PDF → Text)                                 │
│  └─ Converts PDF to plain text                                  │
└─────────────────────┬───────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────────┐
│  Code in JavaScript (Build Request Payload)                      │
│  └─ Formats job description                                     │
│  └─ Formats resume text                                         │
│  └─ Creates JSON request for AI                                 │
└─────────────────────┬───────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────────┐
│  HTTP Request (Groq AI API)                                      │
│  └─ Model: llama-3.3-70b-versatile                              │
│  └─ Analyzes resume vs. job requirements                        │
│  └─ Returns: name, email, phone, score, reasoning               │
└─────────────────────┬───────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────────┐
│  Code in JavaScript (Parse Response)                             │
│  └─ Extracts JSON from AI response                              │
│  └─ Handles errors and edge cases                               │
│  └─ Combines with file metadata                                 │
└─────────────────────┬───────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────────┐
│  Append or Update Row in Sheet (Google Sheets)                  │
│  └─ Matches on: File ID (prevents duplicates)                   │
│  └─ Saves all candidate data and analysis                       │
└─────────────────────┬───────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────────┐
│  Wait (10 seconds)                                               │
│  └─ Rate limiting to respect API constraints                    │
└─────────────────────┬───────────────────────────────────────────┘
                      ↓
                Loop back to process next resume
```

---

<img width="1627" height="514" alt="image" src="https://github.com/user-attachments/assets/9faee1cf-0693-4e90-9576-030624c987dc" />


## 🛠️ Technical Approach

### 1. **API Selection: Groq AI**
**Why Groq?**
- ✅ Free tier with generous limits (30 requests/minute)
- ✅ Fast inference (llama-3.3-70b model)
- ✅ High-quality reasoning capabilities
- ✅ Structured JSON output support

**Alternatives Considered:**
- ❌ Anthropic Claude: Requires paid tier
- ❌ Google Gemini: PDF processing limitations

### 2. **Rate Limiting Strategy**
**Approach:** Sequential processing with delays
- Process 1 resume every 10 seconds = 6 resumes/minute
- Well under the 30 requests/minute limit
- Prevents rate limit errors
- Ensures 100% processing success rate

**Alternative Rejected:** Batch processing (would hit rate limits)

### 3. **Duplicate Prevention**
**Approach:** File ID matching
- Each resume has a unique Google Drive File ID
- Google Sheets node matches on "File ID" column
- If File ID exists → Updates existing row
- If File ID is new → Appends new row
- Prevents duplicate entries when workflow runs multiple times

### 4. **Error Handling**
**JSON Parsing:**
- Handles markdown code blocks from AI (```json...```)
- Strips control characters that break JSON
- Fallback parsing with try-catch blocks
- Graceful degradation if parsing fails

**PDF Extraction:**
- Handles various PDF formats
- Extracts text reliably from diverse resume layouts

### 5. **Scoring Methodology**
**Weighted Evaluation:**
```
Total Score (0-100) = 
  Skills Match (40%) +
  Experience Relevance (30%) +
  Education Alignment (15%) +
  Additional Qualifications (15%)
```

**AI Prompt Engineering:**
- Clear scoring criteria provided to AI
- Structured JSON output format specified
- Detailed reasoning requirement (2-3 paragraphs)
- Ensures consistent, explainable results

---

## 📊 Output Schema

**Google Sheet Columns:**

| Column | Description | Example |
|--------|-------------|---------|
| **Candidate Name** | Full name extracted from resume | "Ayush Raj" |
| **Mobile** | Phone number (if available) | "9389429033" |
| **Email** | Email address (if available) | "ayush@example.com" |
| **Resume Link** | Direct link to PDF in Google Drive | drive.google.com/file/... |
| **Relevance Score** | AI-calculated score (0-100) | 92 |
| **Reasoning** | Detailed explanation of score | "The candidate demonstrates..." |
| **Date Processed** | Timestamp of analysis | "2025-12-31" |
| **File ID** | Unique Google Drive identifier | "1XtEsbBnTq..." |

---

## 🎯 Key Features

### ✅ **Automation**
- Fully automated end-to-end process
- No manual intervention required
- Continuous monitoring via schedule trigger

### ✅ **Scalability**
- Handles 2 resumes (demo) to 100+ resumes (production)
- Rate limiting ensures reliable processing at scale
- Duplicate detection prevents data redundancy

### ✅ **Accuracy**
- Advanced AI model (70B parameters)
- Structured scoring methodology
- Detailed, explainable reasoning for each score

### ✅ **Reliability**
- Error handling for edge cases
- Robust JSON parsing
- Graceful degradation on failures

### ✅ **Maintainability**
- Clean, modular workflow design
- Clear node naming conventions
- Easy to modify scoring criteria or job requirements

---

## ⚙️ Configuration

### Prerequisites
- n8n instance (self-hosted or cloud)
- Google Drive account with API access
- Groq API account (free tier)
- Google Sheets API access

### Setup Steps

1. **Import Workflow**
   - Upload `Resume_Screening_Workflow.json` to n8n

2. **Configure Credentials**
   - Google Drive OAuth2
   - Groq API Key (Bearer token)

3. **Set Folder IDs**
   - Update "Search files and folders" node with your Resumes folder ID
   - Update Google Sheets node with your sheet ID

4. **Customize Job Description**
   - Modify the JD text in "Code in JavaScript1" node
   - Or read dynamically from Job_Description folder (future enhancement)

5. **Adjust Schedule**
   - Configure Schedule Trigger frequency as needed

6. **Test**
   - Execute workflow manually first
   - Verify results in Google Sheet
   - Then activate for automatic runs

---

## 📈 Performance

### Current Configuration
- **Resumes Processed:** 2 (demo)
- **Processing Time:** ~30 seconds total (10s per resume + API time)
- **Success Rate:** 100%
- **API Rate:** 6 requests/minute (safe buffer)

### Production Scaling
- **50 resumes:** ~10 minutes
- **100 resumes:** ~20 minutes
- **Rate limit:** Adjustable via Wait node duration

---

## 🚀 Future Enhancements

### Possible Improvements
1. **Dynamic Job Description Reading**
   - Read JD from Job_Description folder instead of hardcoding
   - Allows easy JD updates without workflow modification

2. **Email Notifications**
   - Send summary email after processing batch
   - Alert on high-scoring candidates (score > 85)

3. **Advanced Filtering**
   - Only save candidates above threshold (e.g., score > 60)
   - Separate sheets for different score ranges

4. **Resume Analytics Dashboard**
   - Average score calculation
   - Top skills frequency analysis
   - Experience distribution charts

5. **Multi-Role Support**
   - Process resumes for multiple job openings
   - Dynamic JD selection based on role tag

6. **Enhanced Error Logging**
   - Separate error tracking sheet
   - Failed resume retry mechanism

---

## 📝 Design Decisions

### Why Sequential Processing?
**Decision:** Process resumes one at a time with delays

**Rationale:**
- Prevents API rate limit errors (30/min limit)
- Ensures 100% success rate
- Simple, reliable implementation
- Acceptable processing time for typical use case

**Trade-off:** Slower than parallel processing, but more reliable

---

### Why Hardcoded Job Description?
**Decision:** Embed JD in workflow for demo

**Rationale:**
- Faster implementation (time-constrained assignment)
- Demonstrates core automation capabilities
- Easy to understand and test
- Production enhancement clearly documented

**Trade-off:** Less flexible, but functionally complete for demo

---

### Why Groq Over Other AI APIs?
**Decision:** Use Groq's llama-3.3-70b model

**Rationale:**
- Free tier with generous limits
- Fast inference speed
- Excellent reasoning capabilities
- Structured output support

**Alternative:** Claude would provide better analysis but requires payment

---

## 🎓 Key Learnings

### Technical Skills Demonstrated
- ✅ Workflow automation (n8n)
- ✅ API integration (Google Drive, Groq AI, Google Sheets)
- ✅ AI prompt engineering
- ✅ Error handling and edge case management
- ✅ Rate limiting strategies
- ✅ Data transformation and parsing
- ✅ JavaScript/JSON manipulation

### Product Thinking
- ✅ Trade-off analysis (speed vs. reliability)
- ✅ Scalability considerations
- ✅ User experience (duplicate prevention, clear output)
- ✅ Pragmatic scope management (hardcoded JD for demo)

---

## 📞 Notes

- **Test Data:** Currently processes 2 sample resumes
- **Job Requirements:** Product Intern at LeadSquared (technical background, AI tools, communication skills)
- **Scoring:** Weighted evaluation across skills, experience, education, and qualifications
- **Reliability:** Tested and verified with 100% success rate

---

## 🏆 Conclusion

This workflow demonstrates a production-ready approach to automating resume screening using modern AI and automation tools. The system is:
- **Functional:** Processes resumes reliably and accurately
- **Scalable:** Can handle large volumes with proper rate limiting
- **Maintainable:** Clear structure with documented design decisions
- **Extensible:** Easy to enhance with additional features

The implementation balances technical sophistication with practical constraints, showcasing both engineering skills and product judgment—key competencies for the Product Intern role.

---

**Submitted by:** [Ayush Raj]  
**Date:** December 31, 2025  
**Assignment:** LeadSquared Product Intern - Automated Resume Screening System
