[README.md](https://github.com/user-attachments/files/34221940/README.md)
# Automated Lesson Plan Generation Workflow (n8n + Google Gemini)

An automation workflow built with **n8n** that receives lesson requirements from a teacher through a **Google Form**, validates them, generates a complete lesson package using **Google Gemini** (lesson outline, student activities, assessment, answer key, and an optional differentiated version), formats everything into a **Google Docs** document, and emails the result to the teacher through **Gmail**. Every submission is also logged in **Google Sheets**.

---

## 1. Project Overview

### 1.1 Workflow Stages

> Note: the numbering follows the workflow diagram. The diagram labels two nodes as "7"; in this document the *AI Answer Key* node is referred to as **7b**.

| # | Node | n8n Node Type | Purpose |
|---|------|---------------|---------|
| 1 | Google Form (Teacher Input) | External (Google Forms) | Teacher enters Subject, Grade Level, Topic, Learning Objective, Lesson Duration, Differentiation Preference |
| 2 | Form Trigger (On Form Submit) | Google Sheets Trigger (Row Added) | Detects the new response and starts the workflow in n8n |
| 3 | Edit Fields (Prepare Data) | Edit Fields (Set) | Extracts and renames form fields into clean, consistent keys |
| 4 | IF (Validate Input) | IF | Checks that all required fields are not empty (AND condition) |
| 5 | AI Lesson Outline | Google Gemini Chat Model + Basic LLM Chain | Generates a structured lesson outline |
| 6 | AI Student Activities | Google Gemini Chat Model + Basic LLM Chain | Creates engaging student activities |
| 7 | AI Assessment | Google Gemini Chat Model + Basic LLM Chain | Generates MCQ, short-answer, and application questions |
| 7b | AI Answer Key | Google Gemini Chat Model + Basic LLM Chain | Creates the answer key based on the assessment |
| 9 | IF (Differentiation?) | IF | Checks the teacher's differentiation preference |
| 10 | AI Differentiated Version | Google Gemini Chat Model + Basic LLM Chain | Adapts activities and assessment for the selected level |
| 11 | Merge (Combine Outputs) | Merge | Combines outputs with or without differentiation |
| 12 | Edit Fields (Final Output) | Edit Fields (Set) | Formats all content into one clear structure |
| 13 | Google Docs (Create Document) | Google Docs | Creates the lesson materials document |
| 14 | Google Sheets (Append Row) | Google Sheets | Saves teacher input for record and logging |
| 15 | Gmail (Request Clarification) | Gmail | Emails the teacher the list of missing fields (validation failed) |
| 16 | Gmail (Send to Teacher) | Gmail | Emails the final document link and a summary |

### 1.2 Data Flow

```
Google Form → Sheets Trigger → Edit Fields → IF (valid?)
                                   │              ├─ True  → Outline → Activities → Assessment → Answer Key
                                   │              │            → IF (differentiation?)
                                   │              │                 ├─ Yes → Differentiated Version ─┐
                                   │              │                 └─ No  ─────────────────────────┤
                                   │              │                                                  ↓
                                   │              │            Merge → Edit Fields → Google Docs → Gmail (Send)
                                   │              └─ False → Gmail (Request Clarification)
                                   └→ Google Sheets (Append Row – logging)
```

---

## 2. System Requirements

### 2.1 Hardware (self-hosted)

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 1 vCPU | 2 vCPU |
| RAM | 1 GB | 2–4 GB |
| Storage | 5 GB free | 10 GB+ (execution history grows over time) |
| Network | Stable internet access (calls Google APIs and Gemini API) | — |

### 2.2 Software

| Component | Requirement |
|-----------|-------------|
| Operating System | Windows 10/11, macOS, or Linux (Ubuntu 22.04+ recommended for servers) |
| n8n | Latest stable version (self-hosted via Docker/npm, or n8n Cloud) |
| Docker + Docker Compose | Recommended installation method |
| Node.js | Only if installing with npm: use a version supported by your n8n release (see the official n8n docs) |
| Browser | Chrome, Edge, or Firefox (for the n8n editor) |

### 2.3 External Accounts and Services

- A **Google account** (Forms, Sheets, Docs, Gmail).
- A **Google Cloud project** with OAuth credentials and these APIs enabled: **Google Sheets API, Google Docs API, Google Drive API, Gmail API**.
- A **Google Gemini API key** from Google AI Studio.

---

## 3. Installation Steps

### Option A: Docker (recommended)

1. Install Docker Desktop (Windows/macOS) or Docker Engine + Compose plugin (Linux).
2. Create a project folder and a file named `docker-compose.yml`:

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://localhost:5678/
      - GENERIC_TIMEZONE=Africa/Cairo
      - TZ=Africa/Cairo
      - N8N_ENCRYPTION_KEY=change-this-to-a-long-random-string
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

3. Start n8n:

```bash
docker compose up -d
```

4. Open `http://localhost:5678` and create the owner account on first launch.

### Option B: npm

```bash
npm install n8n -g
n8n start
```

Then open `http://localhost:5678`.

### Option C: n8n Cloud

Create an account on n8n Cloud and skip the installation. The workflow is imported and configured exactly as described below.

### 3.1 Import the Workflow

1. In n8n, click **Workflows → Add workflow → ⋯ (menu) → Import from file**.
2. Select `workflow/lesson-plan-generator.json` from this repository.
3. The 16 nodes appear on the canvas. Credentials must be reconnected (see Section 4).

---

## 4. Configuration Instructions

### 4.1 Google Cloud OAuth Credentials

1. Open **Google Cloud Console → New Project**.
2. Enable: **Google Sheets API, Google Docs API, Google Drive API, Gmail API**.
3. Go to **APIs & Services → OAuth consent screen**, configure it, and add your account as a **test user**.
4. Go to **Credentials → Create Credentials → OAuth client ID → Web application**.
5. Add the **Authorized redirect URI** shown inside the n8n credential screen (for local use: `http://localhost:5678/rest/oauth2-credential/callback`).
6. Copy the **Client ID** and **Client Secret**.
7. In n8n: **Credentials → Create → Google Sheets OAuth2 API** (repeat for **Google Docs OAuth2 API** and **Gmail OAuth2 API**), paste the ID and Secret, then click **Sign in with Google**.

### 4.2 Gemini API Key

1. Open **Google AI Studio**, click **Get API key**, and create a key.
2. In n8n: **Credentials → Create → Google Gemini (PaLM) API**, paste the key, and save.

### 4.3 Google Form and Response Sheet

1. Create a Google Form with exactly these fields:

| Field | Type | Required |
|-------|------|----------|
| Subject | Short answer | Yes |
| Grade Level | Dropdown | Yes |
| Topic | Short answer | Yes |
| Learning Objective | Paragraph | Yes |
| Lesson Duration | Dropdown (e.g., 30 / 45 / 60 min) | Yes |
| Differentiation Preference | Dropdown (None / Support / Standard / Advanced) | Yes |
| Teacher Email | Short answer (email validation) | Yes |

2. In the form: **Responses → Link to Sheets** to create the response spreadsheet.
3. Create a second sheet (or tab) named `Submissions Log` for node 14 with columns: `Timestamp, Teacher Email, Subject, Grade Level, Topic, Learning Objective, Duration, Differentiation, Status`.

### 4.4 Node-by-Node Configuration

**Node 2 – Form Trigger (Google Sheets Trigger)**
- Event: **Row Added**
- Document: the form's response spreadsheet
- Sheet: `Form Responses 1`
- Poll Time: **Every minute**

> Google Forms has no native n8n trigger, so the response sheet is used as the trigger source. An alternative is an Apps Script `onFormSubmit` function that sends a POST request to an n8n **Webhook** node (see Section 6).

**Node 3 – Edit Fields (Prepare Data)**
Add these fields (Manual Mapping):

| New Field | Expression |
|-----------|-----------|
| `subject` | `{{ $json["Subject"] }}` |
| `gradeLevel` | `{{ $json["Grade Level"] }}` |
| `topic` | `{{ $json["Topic"] }}` |
| `objective` | `{{ $json["Learning Objective"] }}` |
| `duration` | `{{ $json["Lesson Duration"] }}` |
| `differentiation` | `{{ $json["Differentiation Preference"] }}` |
| `teacherEmail` | `{{ $json["Teacher Email"] }}` |

**Node 4 – IF (Validate Input)**
- Combine conditions with **AND**.
- One condition per required field: `{{ $json.subject }}` → *String → is not empty*, and the same for `gradeLevel`, `topic`, `objective`, `duration`, `teacherEmail`.
- **True** output → Node 5. **False** output → Node 15.

**Node 5 – AI Lesson Outline**
- Model: Google Gemini Chat Model (for example `gemini-2.5-flash` or any currently available Gemini model), Temperature `0.5`.
- Prompt:

```
You are an expert curriculum designer.
Create a structured lesson outline.

Subject: {{ $json.subject }}
Grade Level: {{ $json.gradeLevel }}
Topic: {{ $json.topic }}
Learning Objective: {{ $json.objective }}
Duration: {{ $json.duration }}

Return: Lesson Title, Learning Objectives, Materials, Lesson Timeline
(with minutes per section: Introduction, Direct Instruction,
Guided Practice, Independent Practice, Closure), and Key Vocabulary.
```

**Node 6 – AI Student Activities**
- Input: the lesson outline from Node 5.
- Prompt asks for 3–4 engaging, age-appropriate activities (individual, pair, and group), each with instructions, time, and materials.

**Node 7 – AI Assessment**
- Input: outline plus activities.
- Prompt asks for: 5 multiple-choice questions, 3 short-answer questions, and 1 application question, aligned with the learning objective.

**Node 7b – AI Answer Key**
- Input: the assessment from Node 7.
- Prompt asks for correct answers, brief explanations, and a simple scoring rubric for short-answer and application questions.

**Node 9 – IF (Differentiation?)**
- Condition: `{{ $('Edit Fields').item.json.differentiation }}` → *is not equal to* `None`.
- **Yes** → Node 10. **No** → directly to Node 11.

**Node 10 – AI Differentiated Version**
- Prompt adapts the activities and assessment for the selected level (Support = simplified language, scaffolds, sentence starters; Advanced = extension tasks and higher-order questions).

**Node 11 – Merge (Combine Outputs)**
- Mode: **Append** (or *Combine → By Position*). It combines the outputs with or without the differentiated version.

**Node 12 – Edit Fields (Final Output)**
- Builds one field `finalDocument` that combines the sections in this order: Lesson Outline, Student Activities, Assessment, Answer Key, Differentiated Version (if present).

**Node 13 – Google Docs (Create Document)**
- Operation 1: **Create** a document titled `Lesson Plan – {{ topic }} – {{ gradeLevel }}`, in a chosen Drive folder.
- Operation 2: **Update → Insert Text** with `{{ $json.finalDocument }}`.

**Node 14 – Google Sheets (Append Row)**
- Operation: **Append Row** into `Submissions Log`, mapping the teacher's input fields. It runs in parallel right after Node 2/3 so every submission is recorded even when validation fails.

**Node 15 – Gmail (Request Clarification)**
- To: `{{ teacherEmail }}`
- Subject: `Action needed: missing information for your lesson plan request`
- Body lists the fields that were empty and asks the teacher to resubmit the form.

**Node 16 – Gmail (Send to Teacher)**
- To: `{{ teacherEmail }}`
- Subject: `Your lesson plan is ready: {{ topic }}`
- Body: short summary (subject, grade, topic, duration) and the Google Docs link `{{ $json.documentUrl }}`.

### 4.5 Environment Variables (optional, for deployment)

Create a `.env` file (never commit it to Git):

```env
N8N_ENCRYPTION_KEY=your-long-random-key
N8N_HOST=your-domain.com
N8N_PROTOCOL=https
WEBHOOK_URL=https://your-domain.com/
GENERIC_TIMEZONE=Africa/Cairo
```

---

## 5. Execution Guide

### 5.1 Running Locally

1. Start n8n (`docker compose up -d`) and open `http://localhost:5678`.
2. Open the imported workflow and confirm every node shows connected credentials (no red warnings).
3. Click **Execute Workflow** (manual test) or toggle **Active** to run automatically.
4. Submit the Google Form with sample data, for example:

| Field | Sample |
|-------|--------|
| Subject | Science |
| Grade Level | Grade 7 |
| Topic | Photosynthesis |
| Learning Objective | Students explain how plants convert light into chemical energy |
| Duration | 45 minutes |
| Differentiation | Support |
| Teacher Email | teacher@example.com |

5. Within about one minute (polling interval) the trigger fires. Follow the execution live in the n8n canvas.

### 5.2 Expected Results

| Scenario | Result |
|----------|--------|
| All fields filled | Google Doc is created, a row is added to the log, and the teacher receives an email with the document link |
| A required field is empty | The teacher receives a clarification email listing the missing fields, and no document is generated |
| Differentiation = None | Node 10 is skipped and the document contains the standard version only |
| Differentiation ≠ None | The document also contains the differentiated version |

### 5.3 Testing and Troubleshooting

| Problem | Likely Cause | Fix |
|---------|--------------|-----|
| Trigger does not fire | Workflow not active, or wrong sheet selected | Activate the workflow and check Document/Sheet names |
| `401 / invalid_grant` on Google nodes | Expired or unauthorized OAuth | Reconnect the credential; add yourself as a test user |
| Gemini node returns an error | Invalid API key, quota exceeded, or model name unavailable | Check the key, quota, and select an available model |
| Empty fields in AI prompts | Wrong expression names | Check Node 3 output field names (case-sensitive) |
| Merge node waits or outputs nothing when no differentiation | Merge mode/branch wiring | Use **Append** mode and re-test both branches |
| Google Docs content is empty | Update step missing | Ensure the Insert Text operation uses `finalDocument` |

Use **Executions** in n8n to inspect the input and output of every node for any past run.

---

## 6. API Documentation

The project mainly **consumes** external APIs through n8n nodes and does not expose a custom backend. The optional webhook variant is documented below.

### 6.1 External APIs Used

| API | Used By | Purpose | Authentication |
|-----|---------|---------|----------------|
| Google Sheets API v4 | Nodes 2, 14 | Read new form responses, append log rows | OAuth 2.0 |
| Google Gemini API (Generative Language API) | Nodes 5, 6, 7, 7b, 10 | Text generation | API key |
| Google Docs API v1 | Node 13 | Create and update the lesson document | OAuth 2.0 |
| Gmail API v1 | Nodes 15, 16 | Send emails | OAuth 2.0 |

### 6.2 Gemini Request Format (for reference)

```
POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent
Header: x-goog-api-key: <YOUR_API_KEY>
Content-Type: application/json
```

```json
{
  "contents": [
    { "parts": [ { "text": "Create a lesson outline for ..." } ] }
  ],
  "generationConfig": { "temperature": 0.5 }
}
```

### 6.3 Optional Webhook Endpoint (alternative to Sheets Trigger)

Replace Node 2 with an n8n **Webhook** node and call it from Google Apps Script when the form is submitted.

```
POST https://<your-n8n-domain>/webhook/lesson-plan
Content-Type: application/json
```

Request body:

```json
{
  "subject": "Science",
  "gradeLevel": "Grade 7",
  "topic": "Photosynthesis",
  "objective": "Explain how plants convert light into chemical energy",
  "duration": "45 minutes",
  "differentiation": "Support",
  "teacherEmail": "teacher@example.com"
}
```

Responses:

| Status | Meaning |
|--------|---------|
| `200 OK` | Request received, workflow started |
| `4xx` | Invalid request or webhook not active |
| `5xx` | Workflow execution error (check n8n Executions) |

Apps Script example (attached to the form):

```javascript
function onFormSubmit(e) {
  const r = e.namedValues;
  const payload = {
    subject: r["Subject"][0],
    gradeLevel: r["Grade Level"][0],
    topic: r["Topic"][0],
    objective: r["Learning Objective"][0],
    duration: r["Lesson Duration"][0],
    differentiation: r["Differentiation Preference"][0],
    teacherEmail: r["Teacher Email"][0]
  };
  UrlFetchApp.fetch("https://<your-n8n-domain>/webhook/lesson-plan", {
    method: "post",
    contentType: "application/json",
    payload: JSON.stringify(payload)
  });
}
```

---

## 7. Executable Files and Deployment Link

This project is a **workflow automation**, so it is not compiled into an `.exe`, `.jar`, or `.apk`. The deliverable "executable" package consists of the following files:

| File | Description |
|------|-------------|
| `workflow/lesson-plan-generator.json` | Exported n8n workflow (import and run) |
| `docker-compose.yml` | One-command local/server deployment of n8n |
| `.env.example` | Template for environment variables (no secrets) |
| `form/form-fields.md` | Google Form structure |
| `docs/` | Screenshots, workflow diagram, sample generated lesson plan |

To export the workflow: open it in n8n, then **⋯ → Download**. Remove or avoid committing any credentials.

### 7.1 Deployed Version

Choose one deployment path:

**Option 1: n8n Cloud**
Import the JSON, connect credentials, and activate the workflow. Access link: `https://<your-instance>.app.n8n.cloud`

**Option 2: VPS (Docker + HTTPS)**
1. Rent a Linux VPS (2 GB RAM).
2. Install Docker, then copy `docker-compose.yml` and `.env`.
3. Point a domain to the server and put a reverse proxy (Nginx or Caddy) in front with an SSL certificate.
4. Set `N8N_HOST`, `N8N_PROTOCOL=https`, and `WEBHOOK_URL` in the `.env` file.
5. Run `docker compose up -d`.
6. Update the OAuth redirect URI in Google Cloud to `https://<your-domain>/rest/oauth2-credential/callback`.

**Deployment Link:** `https://<your-deployed-n8n-url>`
**Teacher Form Link:** `https://forms.gle/<your-form-id>`
**Demo Video (optional):** `https://<your-video-link>`

### 7.2 Security Notes

- Never publish API keys, OAuth secrets, or the `N8N_ENCRYPTION_KEY`.
- Use HTTPS in any public deployment and enable n8n user authentication.
- Limit Google OAuth scopes to what the workflow needs.

---

## 8. Repository Structure

```
├── README.md
├── docker-compose.yml
├── .env.example
├── workflow/
│   └── lesson-plan-generator.json
├── form/
│   └── form-fields.md
└── docs/
    ├── workflow-diagram.jpeg
    └── sample-output.pdf
```
