---
layout: post
title: "Stop Bolting Permissions and Audit Logs Onto Your AI Agents"
date: 2026-04-09
---

A comment in a recent discussion about healthcare AI strategy cut through the noise:

> "Stop leading with 'which model?' and start leading with harness architecture, context management, permission systems and auditability."

This is the right framing. The model question is mostly solved — GPT-4o, Claude, Gemini are all capable enough for most enterprise tasks. The unsolved problem is the harness: how do you build a system around an AI agent that is _correct_, _auditable_, and _provably secure_ in terms of data access?

Most teams answer this by adding layers on top of their agent framework: RBAC middleware for permissions, structured logging for audit trails, runtime guards for access control. These work, up to a point. But they share a fundamental weakness: they are runtime mechanisms bolted onto a system whose structure was never designed with correctness in mind.

There is a better answer. Make permissions and auditability structural — enforced by the type system, not by middleware you have to remember to configure correctly.

---

## The Permission Problem

Consider a multi-agent healthcare system. You have a clinical guidelines agent, a billing agent, a radiology agent, and an orchestrator. Each agent needs to access patient data, but each should see only what it needs:

- The billing agent needs financials. It should not see clinical notes or social history.
- The clinical guidelines agent needs the clinical picture. It should not see financial records.
- The radiology agent needs imaging metadata. It should not see either.

The standard approach: write a permission layer that checks the agent's role before returning data. Something like:

```python
def get_patient_data(patient_id, agent_role):
    record = db.fetch(patient_id)
    if agent_role == "billing":
        return filter_to_billing_fields(record)
    elif agent_role == "clinical":
        return filter_to_clinical_fields(record)
    # ...
```

This works until it doesn't. The filter function has to be maintained alongside the data model. A new field added to `PatientRecord` is visible to all agents until someone remembers to update every filter. A misconfigured role silently exposes data it shouldn't. And crucially: when a HIPAA auditor asks "prove that your billing agent could not have accessed clinical notes," the answer is "our logs show it didn't" — which is a statement about what happened, not about what was possible.

"Our logs show it didn't happen" is a weaker claim than "our type system made it impossible."

---

## Projections: Permissions as Types

Ruuk's projection system makes data access a compile-time constraint rather than a runtime check.

```ruuk
pub type PatientRecord = {
    id: Guid
    name: String
    dateOfBirth: Date
    clinicalNotes: List<ClinicalNote>
    financials: BillingRecord
    socialHistory: SocialHistory
    imagingMetadata: ImagingMetadata
}

projection BillingView of PatientRecord =
    only { id, name, financials }

projection ClinicalView of PatientRecord =
    only { id, dateOfBirth, clinicalNotes }

projection RadiologyView of PatientRecord =
    only { id, dateOfBirth, imagingMetadata }
```

Now the billing agent's operation signature is:

```ruuk
pub op assessBillingStatus
    payload record: BillingView
    outcomes =
        | BillingComplete of BillingAssessment
        | InsufficientBillingData
        | PayerNotFound of payerId: String
```

`BillingView` is a type. It contains `id`, `name`, and `financials`. It does not contain `clinicalNotes` or `socialHistory` — not because of a runtime filter, but because they are not fields of `BillingView`. An agent that receives a `BillingView` **cannot access clinical notes**. There is no field to access. Writing code that tries to access `record.clinicalNotes` is a compile error.

This changes the answer to the HIPAA auditor. It is no longer "our logs show the billing agent didn't access clinical notes." It is: "here is the source code — `assessBillingStatus` takes a `BillingView`, which by construction does not contain clinical notes. Access was structurally impossible."

That is a proof, not a log entry.

---

## The Audit Problem

The audit problem in AI systems is subtler than it first appears. The obvious part is logging: record what the agent did, when, and with what data. Most frameworks handle this reasonably well.

The harder part is _structural correctness_: not just "what happened" but "prove that what happened was valid." An auditor for a financial AI system doesn't just want to see that the loan approval workflow ran — they want to know that it was _impossible_ for the system to approve a loan without completing the underwriting step, regardless of what the AI decided.

Runtime logs answer "what happened." A type system answers "what could have happened."

Consider a loan origination saga:

```ruuk
pub resource LoanApplication =
    state Submitted
    state DocumentsVerified
    state CreditChecked
    state Underwritten
    state ConditionallyApproved
    state Closed
    state Rejected

pub op underwriteLoan
    payload application: LoanApplication<CreditChecked>
    performs LoanApplication CreditChecked -> Underwritten
    outcomes =
        | UnderwritingComplete of UnderwritingReport
        | UnderwritingFailed of reasons: List<String>
        | RequiresManualReview

pub op approveLoan
    payload application: LoanApplication<Underwritten>
    performs LoanApplication Underwritten -> ConditionallyApproved
    outcomes =
        | Approved of ApprovalTerms
        | Denied of reasons: List<String>
```

`approveLoan` takes a `LoanApplication<Underwritten>`. The state parameter is part of the type. Calling `approveLoan` with a `LoanApplication<CreditChecked>` — skipping underwriting — is a compile error. The type system enforces the regulatory requirement: approval cannot precede underwriting.

When a regulator asks "prove that your system cannot approve a loan without completing underwriting," the answer is the source code. The compiler rejected every version of the codebase where that sequence was possible. There is no log analysis required, no audit trail reconstruction. The proof is structural.

---

## What This Means for AI Agents Specifically

The concern with LLM-based agents is not just that they might do the wrong thing — it is that they might do the wrong thing in ways that are _invisible_ until something goes wrong in production. A prompt-driven orchestrator that skips a step because of model drift doesn't produce an error; it produces a subtly wrong result.

Ruuk addresses this at two levels:

**Data access**: Agents receive typed projections. A clinical AI agent that receives `ClinicalView` cannot leak financial records to its output — the fields don't exist in its input type. This is not a runtime check that the agent respects. It is a structural constraint the compiler enforces.

**Process correctness**: Sagas declare sequencing and compensation as first-class language constructs. The compiler rejects sequences that violate the resource's state machine. An AI orchestrator that tries to run steps out of order isn't producing a runtime error — it's writing code that doesn't compile.

Together these give you something most agent frameworks don't have: a system where the _structure_ of agent behavior is verifiably correct, independent of what any individual LLM call does.

---

## Auditability Falls Out for Free

When permissions and sequencing are structural, the audit log becomes a much simpler document. You're not trying to reconstruct whether the system followed the rules — the type system already proved it had to. The audit log records _what happened_; the type system proves _what was possible_.

This separation matters enormously in regulated industries. An audit that requires log reconstruction is expensive, error-prone, and ultimately unconvincing — logs can be misconfigured, tampered with, or incomplete. An audit that points to a type-checked codebase and says "this is what the system was structurally capable of doing" is a different category of evidence.

---

## The Harness Is the Product

The LinkedIn comment is right. The model question is secondary. The question that determines whether AI is deployable in healthcare, finance, and insurance is: can you prove — not just assert, but _prove_ — that your system respects data boundaries, follows process rules, and leaves a clean audit trail?

Runtime permissions bolted onto a structurally unsound system can't answer that question convincingly. A type system that makes violations impossible can.

That is the harness architecture worth building.

---

_Ruuk is an open source language designed for correct enterprise processes. Projections, typestate, and sagas are all implemented and under active development._
