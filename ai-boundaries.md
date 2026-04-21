# AI Boundaries for Federal Contracting Tools

AI accelerates federal contracting when it assembles data and formats human reasoning. It breaks federal contracting when it originates the reasoning itself. This document says where that line runs and why 1102tools will not cross it.

## The Line

Tools assemble. Humans reason. An AI that pulls BLS wages, matrixes proposal data, or drafts a memo from a contracting officer's formed judgment is doing work the system already tolerates from support contractors and specialists. An AI that reads a proposal and decides what the strengths are, ranks offerors, recommends an awardee, or concludes a responsibility determination is producing the reasoning the FAR places with a named human actor.

The test is simple. If the signer cannot defend every evaluative claim in the final record without pointing back at the tool's output, the tool crossed the line.

## Three Rules

**1. No model reads a proposal.** Proposals are adversarial inputs. An offeror can embed hidden instructions, white text, or metadata that manipulates any model that reads them. OWASP and the UK NCSC both treat this as an unsolved security problem. Tools that need to interact with proposal text should do so deterministically: rule-based extraction, exact-match quote pulling, requirement-to-proposal crosswalks. Not an LLM.

**2. Reasoning originates with the human, not the model.** Drafting assistance is allowed when the evaluator or CO supplies both the finding and the rationale in the prompt, and the tool formats it. "Vendor A proposed a 4-week transition, this is a strength" is not enough; the model will invent the why. "Vendor A's 4-week transition reduces handover risk because the RFP required 8 weeks, rating this a significant strength" gives the tool something to format, not reason from. If the tool generates the why, the tool generated the reasoning.

**3. The record has to be reconstructable.** No AI-assisted workflow that can influence exclusion, evaluation, or award is acceptable without tamper-evident retention of the prompt, the model and version, inputs, outputs, and human edits. GAO reviews the administrative record. COFC reviews the administrative record. A workflow whose record cannot be reproduced is a workflow whose decision cannot be defended.

## Why It Matters Now

Agencies are using commercial LLMs for evaluation-adjacent work. No protest on AI-specific grounds has sustained yet, in part because the one filed was abandoned when the agency attested human evaluators did the work and the protester could not rebut. That will not last. GAO's most prevalent sustain ground in FY2025 was unreasonable technical evaluation. The third was unreasonable rejection of proposal, new to the top three. Both map directly to AI-assisted workflows that flag, rate, or eliminate without human reasoning behind the result.

The real failure mode is not that LLMs write badly. Modern LLMs write persuasively. The failure is that they assert connections between a proposal and a FAR-compliant conclusion that the proposal does not actually support. When a protester compares the evaluation record against the proposal text and finds conclusions nothing in the proposal establishes, GAO sustains. The pattern is familiar. LLMs just produce it faster and more confidently.

## The Principle

Every skill in 1102tools is built so a contracting professional can look at the output, verify the data, apply their own reasoning, and sign their name to a record they can defend. Tools that assemble, extract deterministically, or draft from formed human reasoning are accelerators. Tools that read proposals, originate evaluative judgments, or produce records no one can reconstruct are replacements for authority the FAR places with a human. The first kind makes the workforce faster. The second kind produces outputs no one can sign and mean it.
