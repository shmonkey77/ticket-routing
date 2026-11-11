# AI Ticket Router: Design Write-up

**Author:** Akash Srinivasan
**Role:** Sole designer and engineer
**Stack:** Python, Google Cloud Run Jobs, Cloud Scheduler, Firestore, Gemini on Vertex AI, Terraform, GitHub Actions

> The source code is private company code. This write-up describes the problem, the architecture, and the design decisions, with no proprietary details.

---

## The problem

Every bug report from our users landed in a single Linear triage queue. Someone had to read each one and decide where it belonged: an engineering issue that stays in Linear, or an IT and support request that belongs in Freshservice. Tickets sat in the wrong place until a person had time to sort them, and that slowed down both teams.

Nobody assigned me this. I saw the bottleneck and built a service to route tickets automatically, without giving up the human judgment the process depended on.

## Goals

1. Route each new report to the right team automatically.
2. **Never create a duplicate ticket or take the same action twice**, even if a run fails partway through and retries.
3. **Humans stay in control.** A person's decision always overrides the model.
4. Prove the classifier is accurate *before* it's allowed to touch real tickets.

## Architecture

```
Cloud Scheduler ──► Cloud Run Job (Python)
                        │
                        ├─► Linear API: fetch new triage reports
                        ├─► Firestore ledger: skip anything already handled
                        ├─► Gemini on Vertex AI: classify the report
                        ├─► Act: keep in Linear (engineering) or open a Freshservice ticket (IT)
                        └─► Firestore ledger: record the decision and action
```

- **Scheduled job.** Cloud Scheduler triggers a Cloud Run Job on an interval. Each run is short and stateless, and all state lives in the ledger.
- **LLM classification.** Each report goes to Gemini on Vertex AI with a structured prompt and comes back as a category the code can act on.
- **Firestore ledger.** This is the source of truth for every report the router has seen and every action it took.

## Key design decisions

### 1. Idempotency through a ledger
A scheduled job will eventually crash halfway through a run, and the next run must not open a second Freshservice ticket for the same report. Before acting, the job checks the ledger. After acting, it records the result. Re-running a batch is always safe.

### 2. Humans always win
If a person has already moved or re-labeled a ticket, the router respects that and doesn't overwrite it. The automation handles the obvious cases, and people keep the final say.

### 3. Shadow mode before write access
The router first ran in **read-only shadow mode**. It classified real reports and logged what it *would* have done, without changing anything. That let me compare its decisions with what people actually did before giving it any authority.

### 4. An evaluation harness for the classifier
I built a harness to measure classification accuracy, so prompt or model changes are checked against a measurable baseline instead of judged by eye.

### 5. End-to-end validation in a sandbox
Before production, I tested the full flow against a Linear sandbox, from fetch through classification to action, so the first real run wasn't the first real test.

### 6. Infrastructure as code, with no long-lived keys
Dev and prod environments are defined entirely in Terraform and deployed through GitHub Actions. Authentication uses **keyless Workload Identity Federation**, so no service-account keys are stored anywhere, and secrets are managed outside the repo.

## Tradeoffs I considered

- **Scheduled polling vs. real-time webhooks.** Polling on a schedule is simpler to operate and retry, and a short delay in routing doesn't matter here. Webhooks would cut latency but add more failure modes.
- **LLM vs. keyword rules.** Bug reports are written in free-form language, so rules would be brittle. An LLM handles the variety, and the eval harness and shadow mode manage the risk of a model making mistakes.
- **Automation vs. control.** I chose a conservative design: read-only first, human overrides always respected, and idempotent actions. That makes it safe to roll out gradually instead of all at once.

## What I learned

Calling the model is the easy part. The real engineering is everything around it: making actions safe to retry, keeping humans in control, and measuring accuracy before trusting the output.
