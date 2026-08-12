---
name: job-search-agent
description: Evaluate job postings against candidate criteria (visa/comp gates, role fit), draft tailored resume bullets and cover letters from real proof points, and track applications in a TSV log. Use when the user pastes a job description/URL and wants it gate-checked, scored, or turned into application materials, or wants to check/update their applications tracker.
---

# Job Search Agent

A Claude Code skill for evaluating job postings, drafting tailored application
materials, and tracking applications. Reads candidate data from `profile.json`.

## Modes

Invoke by describing what you want, or naming the mode directly
(e.g. "evaluate this JD" / "run gate-check on this posting").

### 1. gate-check (run first, always)

Input: a job description (pasted text or URL).

Check the JD against `profile.json` -> `must_haves` before doing anything else:
- Does the employer appear to sponsor H-1B / participate in E-Verify?
  (Look for explicit statements. If the JD is silent, flag as "unknown -
  verify manually" rather than assuming yes or no.)
- Does the posted title/duties reasonably map to SOC 13-2051
  (Financial Analysts)? Titles like "Business Analyst," "Data Analyst," or
  "Accountant" may not map cleanly - flag if ambiguous.
- Is the posted or inferable comp at/above `comp_floor`? If not listed,
  say so rather than guessing.

Output: PASS / FAIL / UNKNOWN for each check, with a one-line reason.
If any check is FAIL, stop here and report - do not proceed to evaluate.

### 2. evaluate

Input: a JD that has passed (or been manually overridden past) gate-check.

Score fit 1.0-5.0 (holistic judgment, not a formula) considering:
- Role/skills alignment against `proof_points`
- Seniority match
- Comp vs `comp_floor`
- Company/industry fit relative to `target_roles`

Output a short verdict (3-5 sentences): score, 1-2 strongest proof points
for this role, 1-2 gaps, and a recommendation (apply / pass / apply with
caveats).

### 3. draft

Input: an evaluated JD (score >= 3.0, or user override).

Using `profile.json` proof_points, draft:
- 3-5 tailored resume bullets, prioritized by relevance to this JD, with
  keywords from the JD naturally worked in (never fabricated - only
  rephrase real proof points)
- A short cover letter / outreach note draft

Save output to `outputs/{company}-{role}.md`. Never overwrite an existing
file for the same company+role without asking.

### 4. track

After gate-check/evaluate, append a row to `applications.tsv`:
company, role, score, grade (A: 4.5+, B: 4.0-4.4, C: 3.0-3.9, D/F: <3.0),
status (evaluated/drafted/applied/interviewing/rejected/offer), url,
date_added, notes.

Check for existing rows with the same company+role before adding - update
status instead of duplicating.

## Design principles

- **Human decides, agent analyzes.** Never mark something "applied" unless
  the user confirms they actually applied.
- **No fabrication.** Every resume bullet must trace back to a real
  `proof_points` entry. Rephrasing and keyword injection are fine;
  invented numbers or responsibilities are not.
- **Visa/comp gates are hard stops**, not soft signals - they're cheaper
  to check first than to score a role that's disqualified anyway.
- **Ask before guessing.** If a JD is genuinely ambiguous on sponsorship,
  title mapping, or comp, say "unknown" - don't infer optimistically.
