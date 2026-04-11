---
layout: post
title: "Making Illegal States Unrepresentable: A Response to Paul Vixie"
date: 2026-04-11
---

_A response to "[On the Evolution of Program State](https://queue.acm.org/detail.cfm?id=3799737)" by Paul Vixie, ACM Queue, 2026._

---

Paul Vixie is right. His framing of software safety as a _defined versus undefined program state_ problem is the clearest diagnosis of the root cause I've read. Most safety conversations treat symptoms — buffer overflows, injection attacks, logic errors — without naming what those symptoms have in common. Vixie names it: every one of them is a path by which a program enters a state its programmers never anticipated, and once there, no subsequent operation can be trusted.

I want to engage with his specific concerns and show how ruuk, a language I'm building for stateful process orchestration, approaches the same problem — not through tools bolted on afterward, but at the language design level.

---

## He's Describing the Same Problem from a Different Angle

Vixie's core claim is that software safety is a state-identity problem. A program's state is _defined_ if every variable, effector, and hidden system state sits within the envelope the programmer anticipated and handled. Step outside that envelope, and you're in undefined territory where nothing can be trusted — not just the corrupted variable, but every operation that follows.

This is a stronger claim than "avoid buffer overflows." It encompasses:

- **Memory corruption** — corrupting hidden state like stack frames or heap metadata
- **Modality errors** — not handling all cases of an enumerated return value
- **Data-as-code** — promoting untrusted runtime data into an interpreter
- **Lifetime violations** — operating on a resource after it's been freed or superseded
- **Unhandled failures** — continuing normally after an antecedent operation failed

Vixie's call is to make those paths impossible, or at minimum detectable. Ruuk starts from the same premise, applied to process orchestration: defined program states should be encoded in the type system, and undefined states should not be writable code.

---

## Encoding Valid States in the Type System

Ruuk targets process orchestration — multi-step workflows with external dependencies, business rules, and compliance requirements. This domain has its own catalog of undefined states: workflows that skip mandatory steps, resources operated on after they've moved to a terminal state, partial failures that continue as if they succeeded.

The central design choice in ruuk is **typestate**: encode valid states as types, and bind operations to the states they require. A railroad doesn't prevent derailments by posting "don't leave the track" signs. It uses physical switches that only connect to valid track combinations. Ruuk's type system is those switches.

The `resource` declaration defines a state machine for a domain entity:

```ruuk
pub resource LoanApplication =
    state Draft
    state Submitted
    state Processing
        state DocumentCollection
        state CreditCheck
        state Underwriting
        state ConditionalApproval
        state FinalApproval
    state Closing
    state Terminal
        state Funded
        state Denied
        state Withdrawn
        state Expired
```

A loan application in the `Draft` state has type `LoanApplication<Draft>`. An application in `CreditCheck` has type `LoanApplication<CreditCheck>`. These are distinct types. Operations are typed against the states they require:

```ruuk
op approveFinal (app: LoanApplication<ConditionalApproval>) =
    performs LoanApplication.ConditionalApproval -> LoanApplication.FinalApproval
    outcomes =
        | FinallyApproved of LoanApplication<FinalApproval>
        | ConditionsNotMet of List<Condition>
```

The `performs` clause is not documentation. It's a type annotation. Passing a `LoanApplication<Draft>` to `approveFinal` is a compile error. The sequence of states a loan application must traverse — documents, credit, underwriting, conditional approval, final approval — is encoded in the type signatures of the operations. It cannot be bypassed by any code the compiler accepts.

The set of defined program states for a `LoanApplication` is exactly the enumerated set in its `resource` declaration. Transitioning to an undeclared state is not a runtime error — it's not writable code.

---

## Exhaustive Outcomes: Deliberate Action Made Mandatory

Vixie identifies a subtler path to undefined state: the modality error. A function returns an enumeration with N cases, and the caller handles N-1 of them. The program continues operating on the assumption that the handled cases cover everything, when in fact the unhandled case is the one that just occurred.

This is pervasive. Every function that returns an error code or status enum is an invitation to silently mishandle one of its values.

Ruuk's `outcomes` block closes this gap. Every `op` declares exactly the outcomes it can produce:

```ruuk
pub op runCreditCheck (app: LoanApplication<CreditCheck>) =
    outcomes =
        | CreditApproved of { reports: List<CreditReport>; midScore: FICO }
        | CreditDenied of { reason: CreditDenialReason; reports: List<CreditReport> }
        | BureauError of CreditBureauError
```

The caller handles all three or the code doesn't compile. There's no silent fallthrough. `BureauError` cannot be ignored — not by oversight, not by deadline pressure, not by a junior developer who only read the happy-path documentation. The outcome set is closed and the compiler enforces exhaustive handling.

The same guarantee applies to `match` on union types:

```ruuk
pub type DenialReason =
    | InsufficientCredit of { required: FICO; actual: FICO }
    | InsufficientIncome of { dti: Decimal; maxDTI: Decimal }
    | InsufficientAssets of { required: Money; available: Money }
    | UnsatisfactoryCredit of List<PublicRecord>
    | UnacceptableProperty of String
    | IneligibleProgram of String
```

A `match` on `DenialReason` that omits any variant is a compile error. Vixie's requirement — "corrective action must be deliberate" — is structurally enforced. You can't accidentally not handle a case.

---

## Removing the Data-as-Code Pathway

Vixie's analysis of `eval()`, `popen()`, `system()`, and SQL injection traces them all to the same architectural mistake: the language provides a mechanism to compose a string at runtime and hand it to an interpreter, and that interpreter cannot distinguish programmer intent from attacker-supplied input. The mail slot accepts letters and people with equal indifference.

Ruuk doesn't have `eval()`. It doesn't have `popen()`. There is no string-to-interpreter pathway in the language.

This isn't a feature omission — it reflects a deliberate constraint on how side effects are expressed. In ruuk, operations that interact with external systems are declared as typed `op` entries with named parameters and a specified outcome set. An operation that queries a database takes a `Guid` and an `Int`, not a SQL fragment. An operation that sends a message takes a structured payload type, not a shell command string. There's no ambient interpreter that could reinterpret input as code, because code is never expressed as data.

Vixie describes the safe alternative to `popen()` as accepting an array of argument strings and bypassing the shell. Ruuk's `op` declarations are that alternative, generalized: a structured, typed interface to side effects that requires the programmer to name and type every parameter, with no interpretation layer in between.

"The software engineering community must outright remove or restrict the silliest of dangerous ways to express program logic," Vixie writes. Ruuk removes them by construction.

---

## Failure Continuation: Making Silence Impossible

Vixie makes a point that doesn't get enough emphasis: a program should not continue to operate normally after an antecedent operation completes non-usefully, unless that operation was explicitly optional. Most software fails this test constantly. Error codes go unchecked. Exceptions get caught and logged and swallowed. The program continues on false assumptions about what succeeded, accumulating undefined state with every step.

Ruuk's `|> on` pipeline forces explicit routing of every outcome. The program cannot proceed down the success path without the compiler verifying that the successful outcome was actually received:

```ruuk
runParallelVerification creditApp
|> on AllVerified verifiedResults ->
    moveToUnderwriting creditApp
    |> on ReadyForUnderwriting uwApp ->
        routeForApproval uwApp midScore Underwriter
        ...
|> on CreditFailed _ ->
    LoanDenied of { reason = IneligibleProgram "Insufficient credit"; stage = "Credit Check" }
|> on AppraisalFailed issue ->
    LoanDenied of { reason = UnacceptableProperty issue; stage = "Appraisal" }
|> on TitleFailed _ ->
    LoanDenied of { reason = UnacceptableProperty "Title defects"; stage = "Title Search" }
|> on MultipleFailures _ ->
    LoanDenied of { reason = IneligibleProgram "Multiple failures"; stage = "Verification" }
```

The underwriting path executes only inside the `AllVerified` branch. Every other outcome routes elsewhere. The compiler rejects any code where the failure cases could silently fall through.

For multi-step processes with external commits, ruuk provides `saga` declarations — each forward step paired at declaration time with its compensating action:

```ruuk
pub saga LoanClosing =
    step verifyFinalConditions        compensate noOp
    step prepareClosingDisclosure     compensate deleteClosingDisclosure
    step sendToEscrow                 compensate returnFromEscrow
    step recordDeed                   compensate requestDeedCorrection
    step fundLoan                     compensate reverseDisbursement
    step obtainInsurance              compensate cancelInsurancePolicy
    step completeClosing              compensate noOp
```

If `fundLoan` succeeds and `obtainInsurance` fails, the saga runs `reverseDisbursement`, `returnFromEscrow`, `deleteClosingDisclosure` automatically. The compensation logic lives beside the forward logic in the source — not in a catch block that someone has to remember to keep synchronized with the operations it's meant to compensate. There's no path through loan closing where `fundLoan` has committed and `obtainInsurance` has failed and the system continues as though both succeeded.

This is Vixie's "abort as soon as possible" principle, extended to long-running distributed processes. Sagas don't abort the program, but they drive partial failures to a well-defined compensated state rather than leaving the system stranded in an undefined intermediate.

---

## The Interface Boundary

Vixie recommends inspecting inputs and outputs across abstraction boundaries to account for all knowable side effects. Ruuk enforces this for public operations.

Any `op` declared `pub` — exposed across a module boundary — must have fully annotated parameter and return types. An unannotated parameter on a `pub op` is a compile error. The abstraction boundary carries an explicit type contract, not just documentation that may have drifted from the implementation.

The `performs` clause extends this to state. A caller of `approveFinal` can read from its signature that it requires `LoanApplication<ConditionalApproval>` and produces either `LoanApplication<FinalApproval>` or a `ConditionsNotMet` error. The precondition and postcondition aren't buried in implementation details or a README. They're part of the type — machine-readable and compiler-enforced.

---

## The AI Era Argument

Vixie closes with a point I think about every day: "Today's AI coding assistants augur an era when more software will be created than ever before and by more people and agents than ever before. It's undecided as yet whether this era will be safer or more dangerous than the last."

This is the central design motivation for ruuk. The domain where AI agents are most actively generating code right now is process orchestration — the workflows that connect AI decision-making to consequential real-world actions: loan approvals, healthcare authorizations, financial transactions. These are exactly the workflows where undefined states are most costly and where AI-generated code is most likely to get the sequencing or error handling subtly wrong.

An LLM asked to implement a loan approval workflow in Python will produce code that runs. Whether it correctly implements the state transitions, handles all the error outcomes, or preserves the regulatory requirement that underwriting must precede approval — Python's interpreter won't tell you. Code review is the only check, and code review scales with humans, not with the velocity of AI generation.

An LLM implementing the same workflow in ruuk produces code that either satisfies the type system or doesn't compile. The compiler rejects any version where `approveFinal` is called on `LoanApplication<Draft>`, where `CreditDenied` isn't handled, or where a `pub op` has unannotated parameters. The AI's output has to pass the same checks a human's does. The language is the safety harness — not the reviewer's attention.

That's what I mean by the language being one of Vixie's "automated techniques relying on modern computing that can detect and prevent opportunities for software to enter undefined states." It operates at the earliest possible point: before the program exists.

---

## What Ruuk Doesn't Solve

I should be honest about the limits.

Ruuk doesn't eliminate all undefined states. The `todo` placeholder in operation bodies defers implementation — the skeleton compiles, but the behavior isn't there yet. Business logic errors — computing the wrong interest rate, misreading a regulatory threshold — aren't type errors. The type system proves that the right operations execute in the right sequence on the right types. It doesn't prove those operations compute the right answer.

At the memory level, ruuk's VM is written in Rust, so the heap corruption, use-after-free, and buffer overflow classes that Vixie discusses are addressed by the same ownership and borrow-checking guarantees any Rust program gets. That's a strong foundation — but it's Rust's guarantee, not ruuk's. If you're evaluating the safety story, it's worth understanding which layer owns which properties.

And the language is young. The package manager, FFI, and production-grade tooling are still being built. The claims in this piece are about design properties, not deployment readiness.

---

## Conclusion

Vixie's article matters because it names the problem correctly. Defined program state isn't a property you bolt onto a system with additional tooling. It's a property of the language's design. A language that lets you call a function with the wrong state, ignore an enumerated case, or silently swallow a failure is a language that puts the programmer in charge of a class of correctness properties that the compiler could own.

Ruuk's thesis is that process orchestration — which now covers most enterprise software and almost all AI agent infrastructure — deserves a language that takes this ownership seriously. Typestate, exhaustive outcomes, typed operations, and structured compensation aren't advanced language features. They're the minimum viable design for a domain where undefined program states carry real-world consequences.

Vixie writes that we're unlikely to make differences we cannot name. I agree. Ruuk's contribution is to make the naming precise enough that the compiler can enforce it.
