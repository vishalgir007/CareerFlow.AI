# CareerFlow AI

CareerFlow AI is an n8n workflow that automates LinkedIn job discovery, resume-job matching, and cover letter drafting.

It runs daily, fetches jobs based on your filters, compares each job against your resume using an AI model, stores results in Google Sheets, and sends Telegram alerts for high-scoring matches.

## Project Links

- GitHub: https://github.com/vishalgir007/CareerFlow.AI.git
- n8n Workflow: https://vishalgoswami01.app.n8n.cloud/workflow/a6PwefJMQumcE4Zg

## Workflow Overview

1. Schedule trigger starts the workflow (default: daily at 5 PM).
2. Resume PDF is downloaded from Google Drive.
3. Resume text is extracted from the PDF.
4. Job-search filters are read from a Google Sheet.
5. A LinkedIn search URL is generated from those filters.
6. LinkedIn search results are fetched via HTTP.
7. Job links are extracted from the search-result HTML.
8. Links are split into single items and processed in a loop.
9. Workflow waits 10 seconds between jobs to reduce rate-limit issues.
10. Each job page is fetched and parsed.
11. Job attributes are normalized (description cleanup, job id, apply link).
12. AI compares resume with job description and returns:
    - score (0-100)
    - tailored cover letter
13. AI output is parsed into clean JSON.
14. Job + score + cover letter are appended/updated in output Google Sheet.
15. If score >= 50, a Telegram notification is sent.
16. Loop continues until all jobs are processed.

## Node-by-Node Flow

### 1) Trigger and Resume Input
- `Schedule Trigger`: Runs workflow at fixed time.
- `Download file` (Google Drive): Downloads selected resume PDF.
- `Extract from File`: Extracts text from PDF for AI matching.

### 2) Search Criteria and LinkedIn Fetch
- `Get row(s) in sheet` (Google Sheets): Reads filter config.
- `Create search URL` (Code node): Builds LinkedIn query URL using:
  - keyword
  - location
  - experience level
  - remote mode
  - job type
  - easy apply
- `Fetch Jobs from Linkedin` (HTTP): Loads search result page HTML.
- `Extract Job Links` (HTML): Pulls job posting links.

### 3) Per-Job Processing Loop
- `Split Out`: Expands links array.
- `Loop Over Items`: Processes one job link at a time.
- `Wait`: Adds 10-second delay between requests.
- `Fetch Job Page` (HTTP): Loads a job page.
- `Parse Job Attributes` (HTML): Extracts title, company, location, description, job id.
- `Modify Job Attributes` (Set):
  - Cleans description text
  - Derives LinkedIn job id
  - Builds canonical apply URL

### 4) AI Match and Cover Letter
- `AI Agent`: Sends resume + job description to model and asks for JSON output.
- `OpenAI Chat Model`: Model backend used by AI Agent.
- `Parse AI Output` (Set): Removes markdown code fences and converts output to JSON.

### 5) Persistence and Notifications
- `Append or update row in sheet` (Google Sheets): Upserts job results keyed by `link`.
- `Score Filter` (IF): Checks `score >= 50`.
- `Send a text message` (Telegram): Sends alert for qualified jobs.
- False branch goes back to loop without Telegram alert.

## Data Contracts

### Input Sheet (Search Filters)
Expected columns used by `Create search URL`:
- `Keyword`
- `Location`
- `Experience Level`
- `Remote`
- `Job Type`
- `Easy Apply`

### Output Sheet (Job Results)
Columns written by `Append or update row in sheet`:
- `Title`
- `Company`
- `Location`
- `link`
- `score`
- `description`
- `Cover Letter`

`link` is used as the matching key for append-or-update behavior.

## Prerequisites

- n8n instance (cloud or self-hosted)
- Google Drive OAuth credentials
- Google Sheets OAuth credentials
- OpenAI API credentials (or swap to another supported LLM node)
- Telegram bot credentials + chat ID
- Resume PDF stored in Google Drive

## Setup Steps

1. Import `5_6260258903750091662.json` into n8n.
2. Open every credentialed node and map your own credentials:
   - Google Drive
   - Google Sheets (input and output sheets)
   - OpenAI
   - Telegram
3. In `Download file`, choose your resume PDF from Drive.
4. In `Get row(s) in sheet`, point to your filter sheet.
5. In `Append or update row in sheet`, point to your output tracking sheet.
6. In `Send a text message`, set your Telegram chat ID.
7. Optionally change:
   - schedule time in `Schedule Trigger`
   - delay duration in `Wait`
   - alert threshold in `Score Filter`
8. Run once manually to validate parsing and AI output format.
9. Activate workflow.

## Customization

- Change scoring threshold: edit `Score Filter` right value.
- Use different AI model: replace `OpenAI Chat Model` or adjust model id.
- Improve parser selectors: update CSS selectors in `Parse Job Attributes` and `Extract Job Links` if LinkedIn markup changes.
- Tune request pace: increase wait duration and retry policy to reduce request failures.

## Notes and Limitations

- LinkedIn HTML can change, which may break extraction selectors.
- Aggressive request frequency can trigger blocks/rate limits.
- AI output quality depends on resume clarity and prompt design.
- Always review generated cover letters before applying.
