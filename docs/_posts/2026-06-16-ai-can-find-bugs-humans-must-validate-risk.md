---
layout: post
title: AI Can Find Bugs. Humans Must Validate Risk.
categories: [Product Security]
tags: [ai, appsec, offensive-security, security-review, threat-modeling, ai-agents]
fullview: false
description: A practical framework for validating AI-assisted security reviews before trusting their findings, severity, or remediation advice.
comments: false
---

### Introduction

LLMs are getting better at code.

They can write code, explain code, review code, debug failing tests, propose fixes, and reason across a repository. That matters for software engineering. It matters even more for application security.

Application security is built on code understanding. A security engineer has to understand what the system does, where the trust boundaries are, how data moves, who is allowed to do what, and where an attacker can change the story. If models are becoming better at reading and changing software, then offensive and defensive security will benefit from that same capability.

I use AI models for many parts of security work: threat modeling, secure code review, vulnerability hypothesis generation, safe proof-of-concept design in controlled environments, remediation review, and reporting. They help me move faster. They help me enumerate possibilities. They help me inspect paths I might otherwise delay until later.

But I do not trust an AI security review because the findings sound plausible.

Plausibility is cheap.

Security review is about whether the model found the right risks, understood the system boundaries, and produced evidence that survives adversarial scrutiny.

Scope note: the workflow and examples in this post are generalized. They do not describe any employer, client, production system, internal process, or confidential workflow. Any proof-of-concept discussion here assumes authorized testing in a controlled environment.

### AI Is an Accelerator, Not a Verdict

The most useful way to think about AI in security review is not "replacement."

It is leverage.

Anthropic's guidance on building agents makes a useful distinction between workflows and agents. Workflows follow predefined paths. Agents dynamically choose their process and tool use. The same guidance recommends starting with the simplest solution and adding agentic complexity only when it improves outcomes. Source: [Anthropic, Building effective agents](https://www.anthropic.com/engineering/building-effective-agents).

OpenAI's agent guide makes a similar point: agents can help with complex, multi-step work, but reliable systems need tools, instructions, guardrails, evals, and human intervention for high-impact actions. Source: [OpenAI, A practical guide to building agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf).

That framing applies directly to AppSec.

An AI model can help inspect code paths. It can suggest abuse cases. It can compare a pull request against a threat model. It can draft a finding. It can generate a test for a suspected authorization bypass. It can help explain why a fix is weak.

But the model's output is not the security decision.

My rule is simple:

AI can propose findings. Humans must validate risk.

### The Failure Mode: Confident But Shallow

Bad AI security reviews often fail in a very human-looking way.

They sound reasonable.

They use the right words: privilege escalation, injection, insecure direct object reference, SSRF, sensitive data exposure, weak validation, insufficient authorization.

The problem is that they sometimes produce security language without security proof.

A model may say an endpoint "may lack authorization" without tracing the middleware. It may call something SSRF without proving that user input reaches a server-side fetch. It may flag "sensitive data exposure" without showing that sensitive data actually reaches logs, a response, a queue, or a third-party sink. It may recommend "least privilege" without naming the permission that should be removed.

That is not a security review.

That is a concern list.

Concern lists are useful at the beginning. They are not enough at the end.

For me, a trustworthy AI-generated finding has a minimum bar:

- It names the affected component.
- It explains the violated security expectation.
- It points to concrete evidence.
- It describes who can reach the issue.
- It explains what asset or boundary is affected.
- It proposes a fix that can be tested.

If any of those pieces are missing, I do not reject the finding automatically. But I do downgrade it from "finding" to "hypothesis" until the missing evidence is supplied.

### My Five-Layer Validation Process

When I receive an AI-assisted security review, I validate it in five layers.

#### 1. Did It Understand the Architecture?

Before I care about findings, I ask whether the model understood the system.

Did it correctly identify:

- Trust boundaries.
- User and service identities.
- Data flows.
- External entry points.
- Privileged operations.
- Authorization decisions.
- Secrets and sensitive data.
- Failure modes.
- Deployment assumptions.

If the model misses the shape of the system, every downstream security conclusion is suspect.

For example, if the model reviews an API as if all users belong to one tenant, but the product is multi-tenant, the review is already weak. If it treats an internal service as trusted but that service accepts partner-controlled input, the review may miss the real attack path. If it never identifies where tokens are issued, forwarded, exchanged, and validated, it cannot reliably reason about authorization.

This is why I often ask the model to produce an architecture summary before asking for findings.

The first deliverable is not a vulnerability list.

The first deliverable is a map.

#### 2. Are Claims Separated From Evidence?

A useful AI review should separate what it believes from why it believes it.

I want to see the claim, the evidence, and the reasoning between them.

For example:

| AI Claim | Evidence I Expect | Human Validation |
|---|---|---|
| Missing authorization | Route, middleware, object access path, role check | Test cross-user or cross-tenant access |
| SSRF risk | User-controlled URL reaches server-side request | Confirm egress path and blocked internal targets |
| Secret logging | Sensitive source reaches log sink | Trace value and verify log behavior |
| Privilege escalation | Lower-privilege identity reaches higher-privilege operation | Validate required privilege and impact |
| Insecure deserialization | Untrusted input reaches deserializer with dangerous type behavior | Confirm reachable input and exploit condition |

If I cannot trace a finding to code, configuration, IAM permission, request flow, or deployment assumption, I treat it as an unproven hypothesis.

The model may be right.

But being right by accident is not enough.

#### 3. Did It Miss the High-Risk Boundaries?

This is where human experience matters.

Models are often good at obvious code smells. They can identify missing validation, suspicious string concatenation, unsafe parsing, weak error handling, and common framework mistakes.

But application security often fails at the boundaries.

I look for areas the model may underweight:

- Confused deputy risks.
- Cross-account or cross-tenant access.
- Implicit trust between services.
- Token forwarding and token exchange.
- Authorization bypass through alternate APIs.
- Privileged background jobs.
- SSRF-to-metadata paths.
- Overbroad cloud or IAM permissions.
- Insecure defaults.
- Unsafe deserialization.
- Tenant isolation gaps.
- Logging of secrets or sensitive data.
- Queues, webhooks, and async workflows.
- Business logic that bypasses normal controls.

If the AI review finds minor input validation issues but misses identity, privilege, and tenant boundaries, I do not trust the review yet.

This is one of the most important lessons for new security engineers.

The most dangerous bug is not always the one that looks most technical.

It is often the one that crosses the wrong boundary.

#### 4. Is Severity Based on Exploitability?

Models can confuse "technically true" with "important."

A finding may be real but low impact. Another finding may look boring but create a serious trust-boundary failure.

So I test the severity ranking:

- Can an attacker reach this?
- What privilege is required?
- What asset is affected?
- Does it cross a trust boundary?
- Can it be chained with another weakness?
- Is sensitive data exposed?
- Can the attacker change state?
- Is the exploit path realistic?
- Is there a compensating control?
- Would this matter in the deployed environment?

A finding without exploitability analysis is not a security review.

It is a concern with a severity label.

For offensive validation, this is where a safe proof of concept may help. The goal of a PoC is not to show off. The goal is to test whether the risk is real. In a controlled environment, an AI model can help draft a local test, generate request variants, or write a minimal reproduction that proves reachability without touching production or unauthorized systems.

The PoC is evidence.

It is not the point.

#### 5. Is the Remediation Specific Enough to Verify?

Bad AI remediation often sounds like a bumper sticker:

- Add validation.
- Use least privilege.
- Sanitize inputs.
- Enable encryption.
- Add authorization.
- Improve logging.

Those phrases may be directionally correct, but they are not enough.

Good remediation is specific to the system:

- Add an object-level authorization check in this service method.
- Require tenant ID equality between the authenticated principal and the requested object.
- Remove this cloud permission from this role.
- Reject internal IP ranges before server-side fetch.
- Stop forwarding this token to this downstream service.
- Redact this field before writing structured logs.
- Add an invariant test proving that a user from tenant A cannot read tenant B's object.

I also ask whether the fix addresses the root cause or only the symptom.

A patch that blocks one endpoint but leaves the same operation reachable through another API is not a durable fix. A validation rule that protects the UI but not the API is not a durable fix. A role check that protects synchronous execution but not the background worker may still be bypassable.

The best remediation can be implemented, reviewed, and tested.

### A Practical AI-Assisted Review Workflow

Here is a workflow I would want upcoming security engineers to learn.

First, define the scope.

Tell the model what it is allowed to review, what environment it can reason about, and what it must not do. Security work needs boundaries. AI-assisted security work needs them even more. I also avoid sending secrets, customer data, or unnecessary internal details to any model or tool that does not need them for the review.

Second, ask for architecture reconstruction.

Before findings, ask the model to summarize:

- Entry points.
- Actors.
- Trust boundaries.
- Data stores.
- Sensitive assets.
- Privileged operations.
- External dependencies.
- Security assumptions.

Then correct the model. This is not wasted time. This is how you prevent a review built on a bad map.

Third, ask for a threat model.

Ask for abuse cases by boundary: identity, authorization, tenant isolation, data exposure, injection, async processing, dependency trust, and operational failure. Do not ask only, "find vulnerabilities." That prompt is too shallow.

Fourth, run code review in passes.

Use separate passes for different concerns:

- Authentication and session handling.
- Authorization and object access.
- Input handling and injection.
- Sensitive data flow.
- Secrets and logging.
- SSRF and outbound requests.
- Deserialization and parsing.
- Async jobs, queues, and webhooks.
- Cloud permissions and deployment assumptions.

This mirrors Anthropic's discussion of parallelization and routing patterns, where separate model calls can focus on distinct concerns instead of forcing one pass to do everything.

Fifth, convert findings into a claim-evidence matrix.

No evidence, no finding.

Sixth, ask for misses.

I like to ask a second model pass:

```text
Assume the previous review missed a serious vulnerability.
What would it most likely have missed, and where would you inspect next?
Focus on trust boundaries, identity, privilege, tenant isolation,
and alternate execution paths.
```

This does not magically solve the problem, but it changes the model's posture. Instead of defending its first answer, it has to attack it.

Seventh, validate exploitability safely.

Generate tests or PoCs only inside authorized environments. Prefer unit tests, integration tests, local harnesses, and controlled reproductions. Do not use AI to generate exploit code for systems you do not own or have explicit permission to test.

Eighth, review remediation.

Ask whether the proposed fix removes the class of issue, whether alternate paths remain, and what test would fail before the fix and pass after it.

Ninth, write the report.

AI is useful here too. It can help turn raw notes into a clear finding with impact, evidence, exploitability, affected component, remediation, and validation steps. But the report should only contain what the human reviewer has validated.

### The Agent Risk Comes With the Agent Power

The same capabilities that make AI useful for security also create risk.

OWASP's LLM Top 10 describes prompt injection as a risk where user prompts or external inputs manipulate the model's behavior. It also describes excessive agency as a risk where an LLM-based system has too much functionality, permission, or autonomy. Sources: [OWASP LLM01: Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/) and [OWASP LLM06: Excessive Agency](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/).

That matters for AI-assisted security work.

A security agent may read untrusted issue comments, pull request text, logs, scanner output, bug bounty reports, or exploit descriptions. It may also have access to source code, local tools, test environments, and reporting systems.

That combination needs governance.

Use least privilege. Prefer read-only access when possible. Keep production out of scope. Log tool use. Separate untrusted content from trusted instructions. Require approval for state-changing actions. Keep PoC generation inside safe environments. Treat model output as untrusted until validated.

The agent can accelerate the work.

It should not be allowed to quietly expand the blast radius of the work.

This is also why I prefer evidence-producing workflows over chat-only workflows. A chat answer disappears into interpretation. A test, trace, diff, log sample, permission map, or claim-evidence table gives another reviewer something to challenge.

### What Upcoming Security Engineers Should Learn

The next generation of security engineers should learn how to work with AI without becoming dependent on AI.

That means learning the fundamentals even more deeply:

- How systems are designed.
- How identity works.
- How authorization fails.
- How data crosses trust boundaries.
- How attackers chain small mistakes.
- How to validate reachability.
- How to write useful tests.
- How to explain risk in plain language.
- How to review a fix for bypasses.

AI can help with all of these.

But it cannot care about the outcome.

It cannot own the risk.

It cannot decide what matters to the business.

It cannot replace the human responsibility to ask, "What if this is wrong?"

That question is the center of trustworthy AI-assisted security review.

If this AI review were wrong, how would I know?

If the answer is "I would not know," then the review is not ready to trust.

### Final Thought

LLMs are becoming powerful code workers.

That is good news for security engineers who know how to validate their work.

Use AI to explore more paths. Use it to read more code. Use it to draft tests. Use it to challenge assumptions. Use it to improve reports. Use it to make security work faster and more repeatable.

But do not confuse fluency with proof.

The best AI security reviews are not autonomous verdicts. They are accelerators for expert judgment.

AI can find bugs.

Humans must validate risk.
