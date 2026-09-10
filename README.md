# Blog Automation – TechBliss

An [n8n](https://n8n.io) workflow that fully automates SEO blog writing and publishing for TechBliss: it picks a topic, researches the SERP, drafts and fact-checks an article with LLM agents, generates on-brand images, and publishes the finished post straight to WordPress — once a day, hands-free.

## What it does

Every day at a scheduled time, the workflow:

1. **Picks a topic** from a pre-loaded monthly list of 30 blog titles (one per day of the month, with category and secondary keywords already researched).
2. **Validates input** and normalizes the target WordPress site URL.
3. **Researches the SERP** for the primary keyword via SerpAPI — pulling top competitor titles, common terms, and external link candidates.
4. **Finds internal linking opportunities** by scanning existing WordPress posts for keyword relevance.
5. **Builds an SEO outline** (LLM agent) — title, meta description, slug, H1/H2/H3 structure, FAQ questions, tags, and image prompts.
6. **Drafts the full article** (LLM agent) following the outline.
7. **Fact-checks** the draft (LLM agent).
8. **Applies an SEO pass** — keyword density, on-page optimization (LLM agent).
9. **Scores content uniqueness** and triggers a **rewrite agent** if the article is too similar to existing content.
10. **Runs quality control** and an automated **QC fix pass** if issues are found, then re-validates.
11. **Generates images** with OpenAI (one featured image + in-content images matched to key sections) and uploads them to the WordPress media library.
12. **Resolves WordPress tags/category**, inserts internal and external links, and enforces FAQ/source ordering.
13. **Publishes the post to WordPress** and injects AIOSEO meta fields.
14. **Logs the run** (title, status, URL, etc.) to a Google Sheet.

## Architecture

```
Daily Cron Trigger
      │
      ▼
Blog Titles Data (Monthly) → Flatten Blog Titles → Pick Today's Title
      │
      ▼
Map Title Row + Fixed Config → Input Validation → Validation Gate
      │
      ▼
SERP Fetch (SerpAPI) → Parse SERP Data
      │
      ▼
Get Existing WordPress Posts → Build Internal Link Candidates
      │
      ▼
SEO Outline Agent (Groq) → Parse Outline
      │
      ▼
Draft Article Agent (Groq) → Store Draft
      │
      ▼
Fact-Check Agent (Groq) → Store Fact-Checked Article
      │
      ▼
SEO Optimisation Agent (Groq) → Store SEO Article
      │
      ▼
Semantic Uniqueness Scoring → Uniqueness Gate
      │            │
      │        (if too similar)
      │            ▼
      │   Uniqueness Rewrite Agent (Groq) → Store Final Article
      ▼
Quality Validation → QC Gate
      │            │
      │        (if issues found)
      │            ▼
      │      QC Fix Agent (Groq) → Store QC Fixed → QC Re-Validation
      ▼
Merge After QC
      │
      ▼
Add Money Page Links → Resolve WordPress Tags & Category
      │
      ▼
Generate Featured Image (OpenAI) → Upload to WordPress Media
Generate In-Content Images (OpenAI) → Upload to WordPress Media
      │
      ▼
Prepare WordPress Post → Publish to WordPress → Inject AIOSEO Meta
      │
      ▼
Log Blog Post Activity (Google Sheets)
```

## Requirements

- A running **n8n** instance (self-hosted or cloud) with the **LangChain nodes** (`@n8n/n8n-nodes-langchain`) enabled.
- Accounts and API credentials for:
  - **WordPress** (REST API user with publish + media upload permissions)
  - **SerpAPI** (SERP/competitor research)
  - **Groq** (LLM inference for all writing/QC agents)
  - **OpenAI** (image generation)
  - **Google Sheets** (activity logging)

## Setup

1. Import `Blog_Automation_Techbliss.json` into n8n: **Workflows → Import from File**.
2. Create/attach credentials in n8n for each service above and map them to the corresponding nodes:
   - `SERP Fetch (SerpAPI)` → SerpAPI
   - `Get Existing WordPress Posts`, `Publish to WordPress`, `Upload Image to WordPress Media`, `Upload In-Content Image to WP`, `Inject AIOSEO Meta` → WordPress
   - `Groq (...)` nodes → Groq
   - `Generate Featured Image (OpenAI)`, `Generate In-Content Image (OpenAI)` → OpenAI
   - `Log Blog Post Activity` → Google Sheets
3. Open **`Map Title Row + Fixed Config`** and update the fixed values to match your site:
   - `WordPress Site URL`
   - `Target Audience`
   - `Word Count`
   - `Tone`
4. Open **`Blog Titles Data (Monthly)`** and paste in your month's title list, following the required shape:
   ```json
   {
     "blogs": [
       {
         "category": "Category Name",
         "seo_title": "SEO title for AIOSEO",
         "catchy_title": "Polished headline used as the WordPress post title",
         "secondary_keywords": ["keyword one", "keyword two"],
         "description": "Meta description for AIOSEO",
         "character_count": 120
       }
     ]
   }
   ```
   The workflow picks one entry per calendar day (day 1 → first entry, day 2 → second, etc., wrapping if the list is shorter than the month).
5. Set the desired run time on **`Daily Cron Trigger`** (defaults to 11:00).
6. Activate the workflow.

## Configuration notes

- **Keyword selection**: the primary keyword used for SEO is derived from the first secondary keyword in each title entry (falling back to category, then the full title) rather than the catchy headline itself, so on-page optimization targets a real short-tail search term.
- **Uniqueness & QC loops**: the workflow includes automatic self-correction — if content scores as too similar to existing posts, or fails quality checks, it's routed through a rewrite/fix agent before publishing.
- **Images**: brand colors and style are hard-coded into the image prompts (confident blue `#1E88E5`, warm amber `#F5A623`) for a consistent, on-brand look — update these in the `SEO Outline Agent` prompt if your brand palette differs.
- **Fallback parsing**: if an LLM agent returns malformed JSON, the workflow falls back to a safe default outline so the run doesn't fail outright.

## Disclaimer

This workflow publishes content automatically with no human-in-the-loop approval step. Review the generated posts periodically, and consider adding a manual approval/draft-only step before enabling fully unattended daily publishing on a production site.
