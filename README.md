# DevSynt Hiring Automation Pipeline

An end-to-end automated recruitment intake and ATS scoring system built on **n8n**, integrated with **Gemini AI** for resume evaluation, and **Supabase** for database state management. 

This project handles dual-channel candidate intake (Gmail & Google Forms), normalizes multi-source applicant payloads, calculates explainable candidate match scores using Generative AI, and routes conditional candidate communications via email.

---

## 🏗️ System Architecture & Workflow Overview

The system consists of two primary asynchronous workflows operating in tandem:

### 1. Intake, Normalization & Scoring Workflow
* **Multi-Channel Triggers**: Listens for incoming application emails via the `Gmail Trigger` and structured form submissions via `Google Sheets Trigger`.
* **Data Normalization**: JavaScript Code node standardizes incoming schema differences (email headers vs. form fields) into a unified JSON schema containing candidate contact info, role applied, and resume text.
* **Database Ingestion**: Inserts raw candidate details into Supabase PostgreSQL database with an initial state of `pending_review`.
* **Instant Candidate Acknowledgment**: Sends an automated dynamic HTML intake receipt email back to the applicant.
* **AI ATS Scoring Engine**: Passes candidate resume text to Gemini API (`Analyze document`) to evaluate qualifications against role requirements and extract dynamic numeric scores (0–100) and evaluation summaries.
* **Structured Parsing & DB Update**: JavaScript parser extracts structured JSON outputs from raw Gemini responses and updates candidate records in Supabase with calculated ATS scores and classifications (`Strong`, `Average`, `Weak`).

### 2. Candidate Routing & Communication Workflow
* **Polled Candidate Query**: Scheduled trigger executes `Get many rows` on Supabase to fetch unprocessed candidates (`status = 'pending_review'`).
* **Conditional Routing Switch**: Evaluates candidate classification:
  * **Strong (Score 75–100)**: Routes to Output 1, dispatches Shortlist HTML Email to applicant, and updates status to `'notified'` in Supabase.
  * **Average (Score 50–74)**: Routes to Output 2, alerts Hiring Manager with candidate details for manual review, and updates status to `'manual_review'` in Supabase.
  * **Weak (Score < 50)**: Routes to Output 3, dispatches polite rejection HTML email, and updates status to `'rejected'` in Supabase.

---

## 🛠️ Tech Stack & Integrations

* **Workflow Engine**: Local n8n Instance (v1.x)
* **Intake Channels**: Gmail API (IMAP/Gmail Trigger), Google Sheets API (Google Forms)
* **Database & Persistence**: Supabase (PostgreSQL)
* **AI/LLM Engine**: Google Gemini API (`gemini-1.5-flash` / text analysis)
* **Languages**: JavaScript (ES6+ for n8n Code Nodes), HTML5 (Dynamic Email Templates)

---

## 🗄️ Database Schema (`candidates` Table)

```sql
CREATE TABLE candidates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    email TEXT NOT NULL,
    phone TEXT,
    source TEXT NOT NULL, -- 'email' or 'form'
    role_applied TEXT NOT NULL,
    resume_text TEXT,
    ats_score INT,
    classification TEXT, -- 'Strong', 'Average', 'Weak'
    manager_notes TEXT,
    status TEXT DEFAULT 'pending_review', -- 'pending_review', 'notified', 'manual_review', 'rejected'
    applied_at TIMESTAMPTZ DEFAULT NOW()
);

```
## 🚀 Setup & Installation Instructions
Prerequisites
Node.js installed locally.

Active n8n instance (npm install n8n -g && n8n start).

Supabase Project with the candidates table created.

Google Cloud / Gmail API Credentials and Google Gemini API Key.

##Workflow Import
Clone this repository:

Bash
git clone [https://github.com/your-username/devsynt-hiring-automation.git](https://github.com/your-username/devsynt-hiring-automation.git)
Open your local n8n canvas (http://localhost:5678).

Click Workflows -> Import from File and select workflow.json.

Configure required credentials for:

Gmail OAuth2 Account / API Credentials

Google Sheets OAuth2 Credential

Supabase Service Key / API Credentials

Google Gemini API Key

Enable the workflow toggle switch to activate.

## 🛡️ Error Handling & State ManagementPreventing Duplicate Processing: 

Strict status state transitions in Supabase (pending_review $\rightarrow$ notified / manual_review / rejected) ensure candidate records are processed exactly once per execution cycle.
Payload Resiliency: The normalization node safely handles missing optional fields (e.g., missing phone numbers or header variations) without throw errors.
