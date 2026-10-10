# Dev Scout

**Website:** [omar-khas.github.io/dev-scout](https://omar-khas.github.io/dev-scout/)

**An AI hiring agent that screens developer CVs for small companies.**

Candidates apply by email or web form. Dev Scout reads each CV, finds and analyzes the candidate's GitHub, scores them against the criteria HR set for that role, and puts everyone in a live ranked dashboard with evidence, flags and tailored interview questions. HR gets an email the moment a new candidate is scored.

> The agent screens. HR decides.

Built for **Agents at Work 2026** (Egyptian SMEs).

---

## Try it in under 5 minutes (no setup)

Everything is already running.

1. **Apply as a candidate:** open the [application form](https://khas.app.n8n.cloud/form/dev-scout-apply), type a role (`Software Intern` or `Backend Developer`), and upload a PDF CV. Use [`samples/cv_with_github.pdf`](samples/cv_with_github.pdf) if you don't have one.
   You can also email a PDF CV to the demo inbox (address in the submission form) with the role in the subject, e.g. `Application: Backend Developer`.
2. **Wait about 1 minute** while the agent works.
3. **Open the dashboard:** [devscout-hiring.lovable.app](https://devscout-hiring.lovable.app) and connect with the access key given in the submission form. Your candidate appears in the ranking. Click them to see the full profile.
4. **Create a role:** in **Roles**, click **New role**, pick a template, adjust the weights and save. New applications for that role are scored with your criteria.

---

## What the agent does

```
 Candidate                     Dev Scout agent (n8n)                       HR
 ─────────                     ─────────────────────                       ──
 Web form ──┐
            ├─► Match role ─► Read CV ─► Analyze GitHub ─► Score ─► Save ─┬─► Email alert
 Email ─────┘   (keyword)     (Claude)   (GitHub API)      (Claude    (Data   └─► Live dashboard
                                                            + math)    Table)       (Lovable)
```

1. **Intake.** A candidate applies through the n8n form or emails a PDF CV. The Gmail trigger checks the inbox every minute.
2. **Role matching.** The application is matched to an active role (form field or email subject) and the role's weighted criteria are loaded.
3. **CV reading.** The PDF text is extracted. Hidden hyperlinks inside the PDF (for example a "GitHub" word that links to a profile) are scanned too. Claude turns the CV into structured data: name, headline, skills, experience, education, links, phone.
4. **GitHub analysis (no AI).** The public GitHub API gives repos, languages, stars and merged pull requests. A code step computes:
   - **Consistency:** months with coding activity
   - **Variety:** distinct languages and topics
   - **Collaboration:** pull requests merged into other people's projects
   - **Best repositories:** ranked by stars, forks, lifespan, documentation and license
   - **Originality flags:** mostly forks, many empty repos (framed as "look closer", never as proof)
5. **Scoring.** Claude scores each criterion from 0 to 10 with one evidence sentence, checks every CV skill against GitHub evidence, and writes interview questions specific to the candidate's repos and CV.
   The **final 0 to 100 score is calculated in code** from HR's weights: `round(sum(score × weight) / sum(10 × weight) × 100)`. The AI never invents the final number.
6. **Output.** The candidate is saved (one row per email and role, updated on re-application), HR gets an email with the score and a dashboard link, and email applicants get an auto-reply.

If anything is missing (no GitHub link, GitHub rate limit, unreadable PDF), the agent still saves and scores the candidate and records the problem in `data_gaps` instead of guessing.

## The dashboard

[devscout-hiring.lovable.app](https://devscout-hiring.lovable.app), built with Lovable. It reads everything from the agent through secured webhooks.

- **Applicants:** live ranking per role, filters, auto-refresh every 30 seconds
- **Candidate profile:** score with per-criterion evidence, strengths and concerns, CV timeline, "CV claims vs GitHub evidence", GitHub signals, best repos, originality flags, interview questions with copy buttons, call and WhatsApp buttons, status updates
- **Roles:** create and edit roles and their weighted criteria, with templates and a live preview of each criterion's share of the score

The full prompt used to build it is in [`dashboard/lovable-prompt.md`](dashboard/lovable-prompt.md).

## Business impact (estimates)

| | Manual screening | With Dev Scout |
|---|---|---|
| Time per CV (read CV, check GitHub, write notes) | ~20 minutes | ~1 minute to review the ranked result |
| 50 applicants for one role | ~17 hours of HR time | under 1 hour |
| AI cost per CV | none | a few US cents (2 Claude calls) |

- **Hours saved:** ~19 minutes per CV. n8n's "time saved" tracking is set to 20 minutes per processed CV, so the saving is measured automatically on every run.
- **Cost saved:** fewer interviews with candidates whose CV claims aren't backed by evidence, and less risk of a bad developer hire.
- **Revenue protected:** faster shortlisting means roles are filled sooner, so projects start sooner.

These are estimates based on typical manual screening time, not a measured study.

## Privacy

- Stored: name, email, phone, links and professional information only.
- Never stored: birth date, national ID, home address, photo. Leaving out the birth date also keeps age out of the ranking.
- The form requires consent. Email applicants are told in the auto-reply that their data is stored and how to ask for removal.
- The dashboard requires an access key. Keys and credentials are not in this repo.

## Run your own copy

See [`docs/setup.md`](docs/setup.md). In short: import the two workflows into n8n, create two Data Tables, connect Gmail and Claude, set an access key, publish, and build the dashboard from the Lovable prompt.

## Repository contents

```
workflows/
  dev-scout-intake.json   the agent: form + Gmail triggers, CV reading, GitHub analysis, scoring, notifications
  dev-scout-api.json      webhooks for the dashboard: jobs, candidates, candidate, status, save role
dashboard/
  lovable-prompt.md       the prompt that builds the dashboard in Lovable
docs/
  setup.md                step-by-step self-hosting guide and API reference
samples/
  cv_with_github.pdf      fictional test CV with a GitHub link
  cv_no_github.pdf        fictional test CV without one
  sample-report.json      an example of the report the agent produces
```

## Built with

n8n (workflows, Data Tables, forms, webhooks), Anthropic Claude Sonnet (CV reading and scoring), GitHub REST API, Gmail, Lovable (dashboard).

## Known limitations

- Claude's per-criterion judgment can vary slightly between runs (a few points). The score is a first-pass ranking with written evidence; HR makes the decision.
- Only public GitHub activity is visible. "Not visible on GitHub" is neutral, because much real work is private.
- Without a GitHub token, GitHub allows about 60 requests per hour, roughly 15 candidates per hour. Add a token for higher volume.
- CVs must be PDFs with selectable text. Scanned CVs are saved with a data gap note.

## Author

Omar Khas, Cairo
