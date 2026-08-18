# Operations Automation Lab

Low-code workflow automations modelling lending operations processes, built in n8n.

The workflows here are not toy examples. They model decisions I worked with directly as a Business
Analyst on a digital lending platform: how a loan application is triaged on arrival, and how a
customer complaint is prioritised. The point of building them was to work with automation tooling on
a process I already understand, rather than on a generic tutorial scenario.

---

## Repository contents

| File | What it covers |
|---|---|
| `README.md` | This document — the workflows, the design decisions, and what testing revealed |
| `platform-comparison.md` | Evaluation of n8n against Zapier, with a reasoned recommendation |
| `workflows/` | Exported workflow definitions (JSON) |
| `screenshots/` | Workflow canvases, execution logs and output records |

---

## Why I built this

My background is business analysis on a digital lending platform, where I gathered requirements
across Underwriting, Loan Originations, Risk and Compliance, and translated them into functional
specifications. The analysis and documentation side of that work is familiar to me. What I had not
done was build the automation that a process design eventually becomes.

This lab closed that gap. It also let me test something I had only understood theoretically: how
process rules behave once they are executable rather than written in a specification.

---

## Workflow A — Loan Application Intake and Triage

**File:** `workflows/loan-application-triage.json`

### What it does

| Step | Node | Purpose |
|---|---|---|
| 1 | Form Trigger | Captures the application: applicant, contact, loan amount, purpose, credit score, employment status |
| 2 | Normalise (Set) | Assigns an application ID and submission timestamp |
| 3 | Triage (Switch) | Applies underwriting policy rules to determine the route |
| 4 | Route nodes (Set) | Assigns the route label and the service-level target |
| 5 | Append Row (Google Sheets) | Writes an auditable record of every application and its outcome |

### The triage rules

Rules are evaluated in order. Order is deliberate.

| Priority | Condition | Route | SLA |
|---|---|---|---|
| 1 | Credit score below 550 | Declined — automated | 4 hours |
| 2 | Loan amount above $50,000 | Manual underwriting queue | 24 hours |
| 3 | Everything else | Automated assessment | 48 hours |

### The design decision worth explaining

An application for $80,000 from an applicant with a credit score of 480 satisfies both of the first
two rules. Whichever rule is evaluated first determines the outcome, so the ordering is a policy
decision rather than a technical one.

I placed the credit score rule first. A credit failure is disqualifying regardless of loan size, so
routing that application into a manual underwriting queue would consume an underwriter's time on a
decision that has already been made. Reversing the order would produce a materially different
operational load, and neither ordering is self-evidently correct — it depends on whether the business
wants a human to review high-value declines. That is the kind of ambiguity a specification should
resolve explicitly rather than leave to whoever implements it.

### Testing

Four cases were run deliberately, including the conflict case:

| Test | Loan amount | Credit score | Expected | Result |
|---|---|---|---|---|
| 1 | $20,000 | 700 | Automated assessment | Pass |
| 2 | $80,000 | 700 | Manual underwriting | Pass |
| 3 | $20,000 | 480 | Declined | Pass |
| 4 | $80,000 | 480 | Declined (rule precedence) | Pass |

---

## Workflow B — Complaints Triage

**File:** `workflows/complaints-triage.json`

Applies keyword-based severity scoring to incoming complaints and assigns a response deadline
accordingly.

High-severity terms include *hardship*, *financial difficulty*, *AFCA* and *ombudsman*. These are
not arbitrary. In Australia, a customer disclosing financial hardship triggers specific obligations
for a credit provider, and AFCA is the external dispute resolution scheme. Complaints containing
those signals need a human quickly, so they receive a four-hour target against forty-eight hours for
standard complaints.

---

## What I found while testing

**A silent failure, which was the most useful outcome of the exercise.**

I renamed a column in the destination spreadsheet expecting the workflow to fail. It did not. The
Google Sheets node was set to map columns automatically, so rather than erroring it created a new
column and reported success. The execution was green. The data was wrong.

The same weakness applied elsewhere: had I renamed a field referenced by the Switch node, the
condition would have evaluated false rather than erroring, and every application would have fallen
through to the default route with nothing appearing broken.

This is the failure mode that survives longest in production, because nothing surfaces it until
someone reconciles a report weeks later. I made two changes:

1. Switched the Google Sheets node to explicit column mapping, so a schema change produces an error
   rather than a silent divergence.
2. Added a validation step that throws on missing or non-numeric required fields, so the workflow
   stops at the point of failure instead of writing a corrupt record.

The workflow is now less tolerant and more trustworthy, which is the correct trade for a process
that produces records someone will rely on.

---

## What I would do differently in production

- **Idempotency.** A duplicate form submission currently produces a duplicate record. A production
  version would check for an existing application ID before appending.
- **Thresholds as configuration.** The $50,000 and 550 values are hardcoded into the workflow. They
  are policy settings and belong in a configuration table an operations manager can change without
  editing the workflow.
- **Error handling on the destination.** If the spreadsheet were unavailable, the record would be
  lost. A production version would queue and retry, and alert if the retry failed.
- **Complaint classification.** Keyword matching is crude. It misses paraphrasing and produces false
  positives on quoted text. It is defensible as a first-pass safety net specifically because it errs
  toward escalation, but it is not a substitute for proper classification.
- **Data destination.** A spreadsheet is appropriate for a lab. A real implementation would write to
  the system of record.

---

## Scope

This is self-directed work completed over a single evening to build practical familiarity with
low-code automation tooling. It is not production experience. The value I take from it is a clearer
sense of how process rules behave once they execute, and what breaks when they meet imperfect data.

---

*Tashrif Rayhan — Adelaide, SA*
