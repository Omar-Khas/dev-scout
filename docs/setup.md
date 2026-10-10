# Run your own Dev Scout

About 20 minutes. You need an n8n account (cloud works), a Gmail account for applications, and either n8n AI credits or an Anthropic API key.

## 1. Create the Data Tables

In n8n, open **Data tables** and create two tables with exactly these names and columns.

**`jobs`**

| Column | Type |
|---|---|
| job_id | string |
| title | string |
| email_keyword | string |
| criteria | string (holds JSON) |
| active | boolean |

Add one starter row, for example:

- job_id: `software-intern`
- title: `Software Intern`
- email_keyword: `Software Intern`
- active: true
- criteria:
```json
[
  {"name": "Programming fundamentals", "description": "Solid basics in Python or JavaScript, visible in CV or GitHub", "weight": 3},
  {"name": "Own projects", "description": "Has built real projects of their own, not only tutorials or forks", "weight": 3},
  {"name": "Consistency", "description": "Codes regularly over time, not one burst", "weight": 2},
  {"name": "Collaboration", "description": "Pull requests to others' projects, teamwork, open source", "weight": 1},
  {"name": "Relevant education", "description": "CS or related studies, bootcamps or courses", "weight": 1}
]
```

Later roles can be created from the dashboard's **Roles** page.

**`candidates`**

| Column | Type |
|---|---|
| candidate_id | string |
| job_id | string |
| name | string |
| email | string |
| phone | string |
| github_username | string |
| headline | string |
| score | number |
| score_breakdown | string |
| report | string |
| source | string |
| status | string |
| created_at | date |

## 2. Import the workflows

In n8n: **Create workflow → ⋯ → Import from file**, once for each file in `workflows/`.

After importing, open each workflow and check:

- **Data table nodes** (Get jobs, Save candidate, Read jobs, ...): if a node shows a warning, pick the matching table from the dropdown.
- **Claude nodes** (Read CV, Score and their retries): choose **Use Gateway credits** or **Use my own credential** and paste your Anthropic API key.
- **Gmail nodes** (New CV email, Notify HR, Reply to applicant, Mark as read, Label unmatched email): sign in with the Gmail account that receives applications. In Gmail, create a label `Dev Scout - unmatched` and select it in **Label unmatched email**.
- **Notify HR:** set **To** to the HR inbox that should receive alerts.
- **Config** (Intake): set `dashboard_url` to your dashboard address.
- **Access key** (API): replace `CHANGE_ME` with a long random key. The dashboard uses it as its login.

## 3. Publish

Click **Publish** on both workflows. The form is then live at `https://<your-n8n>/form/dev-scout-apply`, and the inbox is checked every minute.

Optional: in the Intake workflow settings, set **Time saved per execution** to 20 minutes to track hours saved.

## 4. Build the dashboard

Create a Lovable project with the prompt in `dashboard/lovable-prompt.md`. Replace `khas.app.n8n.cloud` in it with your n8n address. Publish it, then connect with your n8n webhook base URL (`https://<your-n8n>/webhook`) and your access key.

## 5. Test

1. Apply through the form with `samples/cv_with_github.pdf`, role `Software Intern`.
2. Within a minute, check the `candidates` table, the HR email and the dashboard.
3. Apply with `samples/cv_no_github.pdf`: the candidate is still scored, with "No GitHub profile found" in data gaps.
4. Email a PDF CV to the Gmail inbox with `Software Intern` in the subject: you get an auto-reply and the candidate appears.

## API reference

All endpoints need `?key=<access key>`. Wrong or missing key returns `401 {"error": "unauthorized"}`. CORS is open.

| Method | Path | Returns |
|---|---|---|
| GET | `/webhook/devscout/jobs` | `{"jobs": [{"job_id", "title", "email_keyword", "criteria", "active"}]}` |
| GET | `/webhook/devscout/candidates?job_id=` | `{"candidates": [...]}` sorted by score; omit `job_id` for all roles |
| GET | `/webhook/devscout/candidate?id=` | the full report (see `samples/sample-report.json`) plus email, phone and status |
| POST | `/webhook/devscout/status` | body `{"candidate_id", "status"}`, status is `new`, `shortlisted`, `interview` or `rejected` |
| POST | `/webhook/devscout/job` | body `{"job_id"?, "title", "email_keyword", "criteria", "active"}`, creates or updates a role |

## Scoring formula

Claude scores each criterion from 0 to 10 with evidence. The final score is computed in the **Final score** Code node:

```
score = round( sum(criterion_score × weight) / sum(10 × weight) × 100 )
```

Example with weights 3, 3, 2, 1, 1 and scores 5, 3, 4, 2, 9: (15 + 9 + 8 + 2 + 9) / 100 × 100 = **43**.
