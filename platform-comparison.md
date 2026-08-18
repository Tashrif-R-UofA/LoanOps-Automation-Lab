# Platform Evaluation — Zapier and n8n

A short comparison of two low-code automation platforms, based on building working flows in both.

The lab workflows in this repository are built in n8n. This note explains why, and what the
alternative looked like in practice. I have written it as an evaluation rather than a tutorial,
because choosing between tools of this kind is usually the more consequential decision — the build
is comparatively easy once the platform is right.

---

## What I built in each

**Zapier.** A two-step automation: new row in a Google Sheet triggers a formatted email
notification. Deliberately simple, because two steps is the ceiling on the free plan.

**n8n.** A loan application intake and triage workflow with conditional routing across three
branches, field validation, and an appended audit record. Documented separately in this repository.

The asymmetry is the finding, not an oversight.

---

## Comparison

| Criterion | Zapier | n8n |
|---|---|---|
| **Pricing unit** | Per task — each action execution is billed. A three-step automation running 100 times costs 300 tasks | Per workflow execution, regardless of how many nodes it contains |
| **Free tier** | 100 tasks per month, two-step automations only | 14-day cloud trial; Community Edition self-hosted with no execution limit |
| **Branching / conditional logic** | Paid plans only | Available throughout, via Switch and IF nodes |
| **Trigger latency** | 15-minute polling on the free plan; faster on paid tiers | Webhook and form triggers fire immediately |
| **Custom logic** | Code steps on paid plans | JavaScript and Python code nodes available |
| **Version control** | No native export to a repository | Workflows export as JSON and can be committed to Git |
| **Hosting** | Vendor-hosted only | Vendor-hosted or self-hosted |
| **Connector breadth** | Very extensive, its principal strength | Substantial, but narrower |
| **Setup effort** | Minimal — an account and a browser | Low on cloud; meaningful on self-hosted |

---

## What the pricing difference actually means

This is the item most likely to be missed at selection and felt later.

Zapier's per-task model means the cost of an automation scales with its complexity. Adding a
validation step, an error branch and a logging step to an existing automation quadruples its running
cost. That creates a quiet incentive to build fewer, thinner automations — precisely the opposite of
what good process design wants, since validation and logging are the steps that make a process
trustworthy.

n8n's per-execution model does not penalise thoroughness. A twelve-node workflow with validation and
error handling costs the same to run as a two-node one.

For a small number of simple integrations, Zapier's model is cheaper and simpler. For operational
processes with real business rules, the incentive runs the wrong way.

---

## Governance and data residency

n8n can be self-hosted, which means customer data need never leave infrastructure the organisation
controls. For a regulated business handling personal and financial information, that is not a
preference — it is frequently the deciding factor, and it is a question that should be raised before
a platform is selected rather than during a security review afterwards.

Zapier is vendor-hosted. That is entirely appropriate for many use cases and removes an operational
burden, but it is a decision about where data flows, and it should be made deliberately.

---

## An incident worth recording

While testing the Zapier automation, it repeatedly sent notifications for the same spreadsheet row
rather than only for new ones.

The cause was trailing empty rows below the data in the source sheet. Zapier's Google Sheets trigger
tracks its position with a cursor, and blank rows disrupt it. The fix was to delete the empty rows
outright — clearing their contents is insufficient, as the rows still exist — and then toggle the
automation off and on to reset the cursor.

The general lesson is more useful than the specific fix: automation triggers depend on assumptions
about the shape of their source data, and those assumptions are rarely documented. A spreadsheet
someone tidies by clearing cells rather than deleting rows is enough to change behaviour. Where a
process depends on a source system nobody formally owns, that dependency should be written down.

---

## Recommendation

**For simple point-to-point integrations across many SaaS applications** — Zapier. The connector
library is its real advantage, and for a two or three step notification or record-copy, the setup
cost is close to zero.

**For operational processes with business rules, validation and audit requirements** — n8n. Branching
is available at no additional cost, the pricing model does not discourage adding controls, workflows
can be version-controlled alongside other code, and self-hosting is available where data residency
matters.

The workflows in this repository use n8n for those reasons.

---

## Limits of this evaluation

This is based on free-tier use over a single evening. It does not assess reliability at volume,
support responsiveness, enterprise administration features, or total cost of ownership at scale —
all of which would matter in an actual platform selection. It is enough to explain a choice between
two tools for a personal lab, and not enough to justify an organisational commitment.

---

*Tashrif Rayhan — Adelaide, SA*
