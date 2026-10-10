Build (or upgrade my existing "Dev Scout" project into) a hiring dashboard called "Dev Scout" for HR people who are NOT technical. It shows job applicants ranked by score and a detailed insight page per candidate.

All data comes from my n8n backend through webhooks. Don't build a database or user accounts. Never hardcode candidates; everything comes from the API.

## Connection settings

- A Settings page with two fields: "n8n base URL" (default `https://khas.app.n8n.cloud/webhook`) and "Access key" (no default, HR types it). Save both in the browser (localStorage).
- First visit, or if any request returns 401: show a simple "Connect to Dev Scout" screen asking for these two values. This acts as a light login.
- Every request adds `?key=<access key>` (or `&key=` if there are other params).

## API

- `GET {base}/devscout/jobs` → `{"jobs": [{"job_id", "title", "criteria": [{"name", "description", "weight"}], "active"}]}`
- `GET {base}/devscout/candidates?job_id=<id>` → `{"candidates": [{"candidate_id", "job_id", "name", "email", "phone", "headline", "github_username", "score", "status", "created_at", "top_skills": [], "strengths": []}]}`, already sorted by score.
- `GET {base}/devscout/candidate?id=<candidate_id>` → full report (shape below).
- `POST {base}/devscout/status` with JSON `{"candidate_id", "status"}`; status is one of `new`, `shortlisted`, `interview`, `rejected`.
- `POST {base}/devscout/job` with JSON `{"job_id"?, "title", "email_keyword", "criteria": [{"name", "description", "weight"}], "active"}` creates a role, or updates it when `job_id` matches an existing role (leave `job_id` out for a new role; it's generated from the title). Responds `{"ok": true, "job": {...}}` or `{"error": "..."}` with status 400. `/jobs` items also include `email_keyword`.

All POST requests send `Content-Type: application/json`.

Show friendly loading and error states. Any field may be missing or empty; never crash, just hide that piece.

Dates come as ISO strings (e.g. `2011-01-27T19:30:43Z`); show them as readable dates. `top_languages` items are objects `{"language", "owned_repos"}`, but also accept plain strings.

## Page 1: Applicants (home)

- Job selector at the top (from `/jobs`, active ones first). Remember the last chosen job.
- Under it, a collapsible "Scoring criteria" card listing the job's criteria with weights as small bars, so HR sees what the ranking is based on, with an "Edit role" button that opens Page 3 for this job.
- A ranked table/list of candidates: rank number, name, headline, score as a big colored badge (80+ green, 60 to 79 amber, below 60 grey; never red), top skills as chips, first strength, status pill, date applied.
- Filters: status, minimum score slider, search by name or skill.
- Summary row above the list: total applicants, average score, number shortlisted.
- "Refresh" button, and auto-refresh every 30 seconds so new applicants slide into the ranking. Briefly highlight newly appeared candidates with a "New" tag.
- Click a candidate to open Page 2.

## Page 2: Candidate insight (`/candidate/:id`)

This URL must work when opened directly from the HR email link.

Report shape:
```json
{
  "candidate_id": "", "job_id": "", "job_title": "", "source": "form | email",
  "generated_at": "", "email": "", "phone": "", "status": "",
  "username": "", "name": "", "public_repos": 0, "followers": 0, "account_created": "",
  "candidate": {
    "name": "", "headline": "", "location": "", "phone": "", "years_experience": null,
    "skills": [""],
    "experience": [{"title": "", "company": "", "start": "", "end": "", "summary": ""}],
    "education": [{"degree": "", "school": "", "year": ""}],
    "links": {"github": "", "linkedin": "", "portfolio": ""}
  },
  "score": 0,
  "criteria_scores": [{"name": "", "score": 0, "weight": 0, "evidence": ""}],
  "strengths": [""], "concerns": [""],
  "summary": {"strongest_domain": "", "top_languages": [{"language": "", "owned_repos": 0}], "top_topics": [""], "owned_repos": 0, "forked_repos": 0, "verdict": ""},
  "metrics": {
    "collaboration": {"merged_prs_total": 0, "merged_prs_into_own_repos": 0, "merged_prs_into_others": 0},
    "total_stars": 0, "variety_distinct_languages": 0, "variety_distinct_topics": 0, "consistency_active_months": 0
  },
  "best_repos": [{"name": "", "description": "", "language": "", "stars": 0, "forks_count": 0, "created": "", "last_push": "", "lifespan_months": 0, "has_description": true, "has_license": true, "topics": [""]}],
  "originality_flags": [{"type": "", "detail": "", "repos": [""]}],
  "claims_vs_evidence": [{"skill": "", "status": "seen_on_github | not_seen", "evidence": [""]}],
  "interview_questions": [{"repo": "", "questions": [""]}],
  "data_gaps": [""]
}
```

Sections, top to bottom:

1. **Header**: name, headline, location, years of experience, job applied for, "Applied via form/email" badge, date, email (with a copy button), phone from `candidate.phone` or `phone` (with a copy button, a "Call" button using `tel:` and a "WhatsApp" button opening `https://wa.me/<number>`; convert Egyptian numbers starting with 0 to the +20 format, e.g. 01012345678 → 201012345678), links as buttons (GitHub, LinkedIn, Portfolio). A status dropdown that calls `POST /status` and updates instantly. Back button to the ranked list.
2. **Score card**: big score out of 100 and a horizontal bar per criterion (score out of 10, weight shown small) with its evidence sentence underneath. Strengths (green ticks) and concerns (neutral grey dots) side by side.
3. **Candidate profile (from CV)**: skills as chips, experience as a vertical timeline, education list.
4. **CV claims vs GitHub evidence**: two columns, "Seen on GitHub" (green check, evidence in small grey text) and "Not visible on GitHub" (neutral grey). Progress bar "X of Y CV skills are visible in public code". Note: "Not visible doesn't mean false. Much real work lives in private or company code."
5. **GitHub signals** (hide the whole block and show a calm card "No GitHub profile found. Ask the candidate for code samples or a portfolio." if `username` is empty):
   - Strongest area + top language chips + verdict.
   - 4 metric cards with plain-English meaning and an info tooltip: Consistency (`consistency_active_months`, "Months with coding activity. Steady work beats one big burst."), Collaboration (`merged_prs_into_others`, "Contributions accepted into other people's projects."), Variety (languages and topics), Recognition (`total_stars`).
   - Small donut: original repos vs forks.
6. **Best repositories**: cards (name, description, language, stars, forks, "Active for N months", last updated, Documented/Licensed badges) and a bar chart of stars. Numbers with thousands separators.
7. **Look closer**: if `originality_flags` is empty, green card "No originality concerns found." Otherwise amber cards with friendly titles (mostly_forks → "Mostly copies of other projects", many_empty_repos → "Many empty projects", single_commit_dump → "Uploaded all at once"). Always add: "These are prompts to ask about, not proof of copying." Never use red or words like cheating or plagiarism.
8. **Interview questions**: grouped by repo, each with a checkbox (saved in the browser per candidate) and copy button, plus "Copy all".
9. **Data gaps**: small grey note if not empty: "Some data couldn't be fetched, so a few numbers may be lower than reality."
10. "Print / Save as PDF" button for a clean printable version.

## Page 3: Roles (`/roles`, linked in the top navigation)

This is where HR creates and edits job roles and the criteria candidates are scored against. Keep it very simple and friendly.

- A list of all roles (from `/jobs`): title, Active/Paused badge, number of criteria, "Edit" button, and a "Copy apply link" button. The apply link is `https://khas.app.n8n.cloud/form/dev-scout-apply?position=<URL-encoded title>`; put that base form URL in Settings as "Application form URL" with this as the default.
- A "New role" button.
- Role editor (for new or existing roles):
  - Title (required), e.g. "Frontend Developer".
  - Email keyword: defaults to the title. Helper text: "Candidates who email a CV with this in the subject are matched to this role."
  - Active toggle. Helper text: "Paused roles stop accepting applications."
  - Criteria list: each row has Name, Description (one line) and Weight (a 1 to 5 stepper or small slider). Add row, remove row, and drag or up/down buttons to reorder. At least one criterion is required.
  - Live preview of weights as percentages ("Backend skills: 30% of the score") so HR understands the effect.
  - "Start from a template" dropdown with 3 presets that fill the criteria: Intern (Programming fundamentals 3, Own projects 3, Consistency 2, Collaboration 1, Relevant education 1), Backend Developer (Backend skills 3, Real projects 3, Code quality 2, Collaboration 1, Experience 1), Frontend Developer (UI skills 3, Real projects 3, Design sense 2, Collaboration 1, Experience 1). HR can edit after applying.
  - "Save role" calls `POST /devscout/job` (include `job_id` when editing). On success show a toast "Role saved" and return to the list; on error show the error message from the response.
- Note at the bottom: "New criteria apply to new applications. Existing candidates keep the score they got."

## Style

Calm, friendly, non-technical. Light theme with dark mode toggle, soft cards, rounded corners, one accent color (indigo). Explain metrics in plain words, no jargon. Works on a phone.

