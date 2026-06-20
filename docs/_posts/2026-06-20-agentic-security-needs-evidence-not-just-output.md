---
layout: post
title: Agentic Security Needs Evidence, Not Just Output
categories: [Product Security]
tags: [ai, ai-agents, appsec, security-engineering, validation, governance]
fullview: false
description: Why LLM-backed security workflows need reproducible evidence, controlled tools, and human accountability before their findings can be trusted.
comments: false
---

### Introduction

Security teams are starting to use LLMs for more than isolated tasks.

At first, the model helps summarize scanner output or explain a code path. Then it starts reading the diff, checking the threat model, generating a test, drafting a finding, and reviewing the fix. At that point, the important question is no longer "can AI find bugs?"

The better question is: what kind of system do we need around the model so its work can be trusted?

That is where a lot of the current conversation still feels too shallow. We talk about AI security tools as if the model is just a smarter scanner or a better assistant inside the same old workflow. But the workflow itself is changing.

The future is not a pile of AI features attached to security tools.

The future is evidence-producing security workflows where LLMs can reason, propose, and draft, but the system around them verifies, records, and controls what is accepted as true.

Put more simply: LLMs propose; deterministic systems dispose.

That is the organizing principle I keep coming back to. The model can suggest that something is vulnerable. The system has to prove it, reject it, or escalate it.

Scope note: the examples in this post are generalized and hypothetical. They do not describe any employer, client, production system, internal process, or confidential workflow. I am mostly talking about internal application security workflows: code review, threat modeling, scanner triage, safe local validation, remediation reporting, and fix verification. External red-team or live-environment testing needs a much stricter approval chain and risk model.

### From Tools to Workflows

Traditional security work often moves through separate tools and separate moments.

A scanner finds something. A reviewer checks it. A threat model sits in a document. A proof of concept is written somewhere else. A ticket is created. A fix lands. Someone tries to remember whether the original issue is actually gone.

The human carries a lot of the context between those steps.

Agentic workflows change that shape.

An LLM-backed workflow can read the code change, pull relevant documentation, inspect prior findings, run a scanner, generate tests, draft a proof of concept in a local environment, summarize evidence, and create a remediation ticket. It can also come back after the fix and rerun the original validation.

That is useful. It is also risky if the organization treats the model's answer as the security decision.

The model can help move context across the workflow.

It should not become the only source of truth.

### The Finding Is Not the Evidence

Security teams already understand this with traditional scanners.

A SAST finding is not automatically a vulnerability. A dependency alert is not automatically an emergency. A DAST result is not automatically exploitable in the real system. These are leads. They need validation.

LLM output should be treated the same way.

If an agent says:

```text
This endpoint appears vulnerable to object-level authorization bypass.
```

that is a hypothesis.

It becomes a stronger finding when the workflow can show:

- The affected route.
- The authorization rule that should apply.
- The role or tenant boundary being crossed.
- The exact request used to test it.
- The response or side effect that proves impact.
- The test or proof of concept that can be rerun.
- The commit, model, prompt, tool versions, and environment used during validation.

The difference matters.

Without evidence, the team is reviewing a model's confidence. With evidence, the team is reviewing a security claim.

### Where LLMs Help Most

LLMs are useful when the work requires context, synthesis, and judgment.

They can help with:

- Reading a large pull request and identifying changed trust boundaries.
- Comparing a route against similar routes in the same codebase.
- Turning a threat model into concrete review questions.
- Explaining why a scanner finding may or may not be reachable.
- Generating negative tests for authorization and input validation.
- Drafting a finding from collected evidence.
- Reviewing whether a proposed fix addresses the abuse case.

This is why agents are interesting for security. They can connect pieces that used to sit in different places.

Anthropic's agent guidance makes a useful distinction between workflows and agents. Workflows follow predefined code paths, while agents dynamically choose their process and tool use. Anthropic also recommends starting with the simplest solution and adding agentic complexity only when it is needed. Source: [Anthropic, Building effective agents](https://www.anthropic.com/engineering/building-effective-agents).

That distinction is important for security.

Some parts of the security process benefit from model judgment. Other parts should stay boring, repeatable, and controlled.

### Where the System Must Prove

Exploit validation is the clearest example.

An agent may be able to write a proof of concept. It may understand the code path, generate an HTTP request, and explain the expected impact. That is valuable creative work.

But validation should not depend only on the same model saying "this worked."

Validation should produce an artifact:

- HTTP transcript.
- Test output.
- Screenshot.
- Stack trace.
- Database state change.
- Log entry.
- Exit code.
- Reproducible script.

The model can propose the test. The harness should run it. The system should capture the result.

This is the boundary that matters:

- Let the LLM reason where reasoning helps.
- Use deterministic code where repeatability matters.
- Require human review where accountability matters.

There also has to be a third state: inconclusive.

Real findings are not always clean pass or fail. A proof of concept may work in one environment but not another. It may require a specific user role, tenant configuration, feature flag, dependency version, or deployment path. A result may prove reachability but not impact. A scanner finding may look exploitable until a downstream control blocks it.

The pipeline should not hide that complexity. If validation is inconclusive, the finding should move to human review with the evidence collected so far. It should not be silently dropped, and it should not be upgraded into a confirmed vulnerability just because the model can explain a plausible exploit path.

The word "deterministic" can be misleading if we apply it to the whole workflow. LLM systems are not deterministic in the same way a normal script is. Even with pinned versions and stable prompts, behavior can vary.

The better goal is reproducibility and auditability.

Can the team rerun the proof of concept? Can the reviewer see what the agent read and what it ran? Can the fix be tested against the original abuse case? Can the organization explain why it accepted or rejected the finding?

If the answer is no, the workflow is not ready to carry serious security decisions.

### Blast Radius Has to Be a Hard Control

Any workflow that lets an agent write and execute exploit code needs a strict blast-radius boundary.

This should not be treated as a nice-to-have engineering practice. It is a risk-control requirement.

Agent-initiated exploit execution should run only against disposable, network-isolated infrastructure created for that validation run. Not shared staging. Not a test tenant connected to real customer data. Not a third-party API that happens to have "sandbox" in the name but still sends real messages or creates real transactions.

The hard controls should include:

- Ephemeral targets that are created for the run and destroyed after it.
- No access to production data.
- No access to shared staging environments unless explicitly approved for that test.
- Default-deny network egress.
- Strict resource limits.
- No real third-party side effects.
- Secrets scoped to the validation environment only.
- Full capture of commands, requests, responses, and tool calls.
- Human approval before any test that touches a non-disposable environment.

This is where many demos quietly cheat. They show the agent writing a proof of concept, but they do not show the isolation model. For a real security program, the isolation model is part of the product.

### A Practical Shape

A useful agentic security workflow might look like this.

First, ingestion is deterministic.

The system gathers the pull request, relevant files, scanner configuration, prior threat model, authorization rules, dependency information, and test environment details. This should be code, not improvisation.

Second, the model reasons over the context.

It identifies changed trust boundaries, likely abuse cases, and candidate findings. It may compare the implementation against known security rules or similar code paths.

Third, scanner output is treated as input, not truth.

Semgrep, CodeQL, dependency scanners, DAST tools, or custom checks can provide useful signals. The agent can cluster and explain them, but the workflow should preserve the raw scanner output so a human can inspect it.

Fourth, validation happens in a controlled environment.

The model may write a test or proof of concept, but the execution happens in a sandbox. The sandbox has clear network rules, resource limits, secrets boundaries, and teardown behavior.

Fifth, evidence is captured.

The result should not be a paragraph saying "the exploit succeeded." It should be a transcript, test log, screenshot, or other artifact tied to the finding.

Sixth, reporting is generated from evidence.

The model can draft the ticket or report, but it should draft from the evidence package, not from memory.

Seventh, fix verification reruns the same test.

The finding should not close because the code changed or the ticket moved. It should close because the original abuse case no longer works, or because a human risk owner accepts the remaining risk.

Eighth, escalation and kill-switch behavior is documented.

If the agent attempts a blocked action, sees unexpected sensitive data, produces repeated inconclusive validations, or starts behaving differently after a model or tool update, the workflow needs a way to pause autonomous execution. That pause should be available at the pipeline level, not buried inside one tool or one prompt.

Every serious pipeline needs a stop button.

### A Worked Example

Architecture diagrams are easy to agree with. The harder question is what the handoffs look like when a real finding moves through the system.

Here is a simplified hypothetical example.

A pull request changes an endpoint:

```text
GET /api/orders/{orderId}
```

The detection agent has access to code search, scanner output, and a threat model store. It does not start by guessing from memory. It pulls evidence first.

```json
{
  "tool": "scanner_lookup",
  "input": {
    "path": "api/routes/orders.py",
    "rule": "missing-object-authorization"
  }
}
```

The scanner result points to an order lookup by `orderId`. The agent then reads the handler and checks the threat model for the service.

```json
{
  "tool": "threat_model_lookup",
  "input": {
    "service": "orders-api"
  }
}
```

The threat model says the route crosses an authenticated-user to orders-database boundary and handles billing-related customer data. The agent also inspects the authentication middleware and notices a possible signature-verification weakness in the way test tokens are accepted.

In CWE terms, this hypothetical chain maps to two familiar classes: improper verification of a cryptographic signature and authorization bypass through a user-controlled key. MITRE describes those as [CWE-347](https://cwe.mitre.org/data/definitions/347.html) and [CWE-639](https://cwe.mitre.org/data/definitions/639.html). The exact labels matter less than the chain: weak token trust plus missing object-level authorization can turn a local bug into cross-user data access.

The detection agent emits a structured hypothesis:

```json
{
  "finding_id": "finding-001",
  "type": "chain",
  "components": [
    {
      "weakness": "improper-signature-verification",
      "location": "auth_middleware.py"
    },
    {
      "weakness": "missing-object-authorization",
      "location": "orders.py"
    }
  ],
  "hypothesis": "A forged test token plus direct order identifier access may allow cross-user order access.",
  "confidence": "unvalidated",
  "data_class": "billing-related customer data"
}
```

The important field is `confidence`.

The detection agent is allowed to write `unvalidated`. It is not allowed to promote the finding to `validated`. That transition belongs to the validation harness.

The orchestrator then creates a disposable target:

```json
{
  "tool": "sandbox_provision",
  "input": {
    "image": "orders-api:pull-request-build",
    "network": "isolated",
    "ttl_seconds": 600
  }
}
```

The model can draft a proof-of-concept test, but it does not run against arbitrary infrastructure. It runs only inside the sandbox, through a narrow execution tool.

```json
{
  "tool": "run_in_sandbox",
  "input": {
    "sandbox_id": "sandbox-001",
    "script": "poc_finding_001.py"
  }
}
```

The grading rule is defined outside the model:

```python
def grade(result):
    if result["exit_code"] != 0:
        return "fail"
    if result["observed_cross_user_access"] is True:
        return "pass"
    return "inconclusive"
```

If the grade is `pass`, the harness writes an evidence package: test script hash, sandbox image digest, request and response transcript, timestamp, tool versions, and the grading result. Only then can the finding move from `unvalidated` to `validated`.

If the grade is `inconclusive`, the finding goes to a human review queue with the partial evidence. It does not disappear. It also does not become a confirmed vulnerability just because the model can tell a convincing story.

Now the ticketing step can happen.

The report agent drafts from the evidence package, not from its own memory of the earlier reasoning. That keeps the report tied to what was actually proven.

```json
{
  "summary": "Possible cross-user order access in orders API",
  "severity": "pending-human-review",
  "evidence": [
    "request-response-transcript.json",
    "poc-script-hash.txt",
    "sandbox-build-digest.txt"
  ],
  "required_review": "security-signoff"
}
```

When the fix lands, the workflow reruns the same proof-of-concept against the patched build. The finding closes only if the original abuse case no longer works, a compensating control is documented, or a human risk owner accepts the residual risk.

The thing to notice is the asymmetry.

The LLM never gets to be both author and judge of the same claim. It can write the test, but the harness grades it. It can draft the report, but only from evidence. It can propose severity, but a human or policy gate decides what enters a compliance-driven SLA.

That is the engineering content behind the phrase "LLMs propose; deterministic systems dispose." It is not a model setting. It is a workflow design choice, enforced by which tools each stage can and cannot use.

### The Evidence Package

This is the part I think security teams should make explicit.

Every accepted AI-assisted finding should have an evidence package.

At minimum, it should include:

- Finding title.
- Affected asset, route, function, dependency, or configuration.
- Security property violated.
- User role or trust boundary involved.
- Steps to reproduce.
- Validation artifact.
- Business impact.
- Suggested fix.
- Fix verification method.
- Human reviewer decision.
- Human attestation or sign-off.
- Known assumptions.

For higher-risk findings, add:

- Sandbox identifier.
- Tool versions.
- Model and prompt version.
- Source commit.
- Test command.
- Network scope.
- Approval record for any sensitive action.
- Escalation history.

This is not bureaucracy for its own sake.

It is what lets a team trust the workflow without pretending the model is always right.

It also answers the audit question.

If a finding feeds a compliance-driven remediation SLA, someone has to own the claim. The harness can produce evidence, but the harness is not the accountable party. A human reviewer, risk owner, or designated security role should attest that the finding is valid enough to enter the remediation process.

That sign-off does not have to mean the human manually redid every step. It means the organization has decided what evidence is sufficient, who is allowed to accept it, and when the finding must be escalated.

The same rule applies when closing a finding. "The agent says it is fixed" should not close the issue. Either the original validation no longer reproduces, the compensating control is documented, or an accountable person accepts the residual risk.

### How to Know It Is Working

A workflow like this should be measured like any other security control.

Useful metrics include:

- False positive rate against accepted findings.
- False negative rate against a labeled eval set.
- Percentage of findings marked inconclusive.
- Percentage of inconclusive findings later confirmed by humans.
- Mean time to validate a candidate finding.
- Mean time from fix to verified closure.
- Percentage of findings requiring human override.
- Percentage of reports with complete evidence packages.
- Number of blocked or escalated agent actions.
- Cost per validated finding.
- Analyst hours saved per validated finding.

These numbers will not be perfect at first. That is fine. The point is to avoid running the system by vibes.

The cost question matters too.

Agentic workflows can become expensive quickly. A single candidate finding may trigger multiple model calls, tool runs, sandbox builds, test executions, retries, and eval checks. That cost may be worth it if it reduces analyst toil, finds issues earlier, verifies fixes faster, or improves evidence quality. But the tradeoff should be measured.

The honest business case is not "AI makes security free."

It is "we are trading some analyst time for compute, infrastructure, and engineering investment, and here is how we know whether that trade is working."

### Evals and Feedback Are the Safety Net

Once a security workflow depends on prompts, tools, models, and agent behavior, the workflow itself needs testing.

OpenAI's agent guide emphasizes clear structured instructions, well-defined tools, layered guardrails, standard security controls, and human intervention for high-risk actions. It also connects human intervention to a stronger evaluation cycle as teams uncover failures and edge cases. Source: [OpenAI, A practical guide to building agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf).

For security teams, that means building evals that look like real security work.

Examples:

- Known vulnerable pull requests that should be flagged.
- Known safe pull requests that should not create findings.
- Authorization bugs with different roles and tenants.
- False positives from past scanner runs.
- Patches that fix the symptom but not the root cause.
- Prompt injection attempts in commit messages, issues, comments, scanner output, or documentation.

The workflow should run against this set when something important changes:

- Prompt update.
- Model update.
- Tool schema change.
- Scanner version change.
- New MCP server.
- New sandbox behavior.
- New output format.

Without evals, a team is changing the security pipeline by feel.

That may be fine for a prototype. It is not enough for a workflow that affects production risk decisions.

Evals should also improve over time.

Every false positive, false negative, weak proof of concept, bad severity call, missed tenant boundary, rejected report, and human override should become a candidate eval or playbook update. Otherwise the workflow will slowly drift away from reality.

This is the continuous-improvement layer. It is not glamorous, but it is where the system either gets better or quietly rots.

### Prompt Injection Is a First-Class Risk

There is another angle teams can miss: the agentic security workflow becomes its own attack surface.

The pipeline reads untrusted content.

It may read:

- Source code.
- Commit messages.
- Pull request descriptions.
- Issue comments.
- Scanner output.
- Web responses.
- Logs.
- Documentation.
- Uploaded files.

Some of that content may contain malicious instructions.

OWASP describes prompt injection as a risk where direct or indirect inputs alter a model's behavior or output in unintended ways. Indirect injection is especially relevant here because the malicious instruction can sit inside content the model later reads. Source: [OWASP LLM01:2025 Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/).

For this kind of pipeline, prompt injection is not a side issue. It is one of the highest-risk design concerns.

A malicious commit message, code comment, issue description, scanner output, or HTTP response could try to influence the agent's next action. During exploit validation, that risk becomes sharper because the agent is reading live responses and deciding what to do next.

The mitigation stance should be explicit:

- Treat retrieved content as untrusted.
- Tool outputs are data, not instructions.
- Prefer structured tool returns over raw text when possible.
- Keep raw untrusted text out of planning context when it is not needed.
- Sanitize or summarize untrusted content before re-entering the reasoning loop.
- Separate instructions from data.
- Limit tool permissions.
- Require approval for sensitive actions.
- Avoid giving one agent private data access, untrusted input, and external communication without strong boundaries.
- Log tool calls and blocked actions.
- Redact secrets before model context when possible.
- Keep destructive actions out of autonomous paths.

Security teams should threat model their AI security tooling with the same seriousness they apply to product systems.

An agent that reviews vulnerable code can also be influenced by vulnerable code comments.

That is not a reason to avoid agents. It is a reason to design them like real systems.

### The New Skill

The skill gap is not only "learn how to prompt."

The deeper skill is knowing how to design the boundary between model reasoning and system verification.

Security engineers will need to understand:

- When an LLM is useful as a reasoning layer.
- When a deterministic check is better.
- How to design tools the model can use reliably.
- How to define structured outputs that downstream systems can trust.
- How to sandbox risky validation.
- How to create evidence packages.
- How to build eval sets.
- How to review agent output critically.
- How to secure the workflow itself.
- How to keep human accountability clear.
- How to keep traditional vulnerability fundamentals sharp enough to challenge the agent.

This is still security work.

It just has more systems engineering inside it.

The best security engineers will not be the people who blindly trust an agent or reject agents out of discomfort. They will be the people who can build workflows where agents help, evidence proves, and humans remain responsible for the decision.

### What This Means for Teams

This is not only an individual skill change. It changes the shape of the team.

If triage becomes more automated, junior analysts still need a path to learn the fundamentals. They should not spend their first year only approving or rejecting agent output. They need to understand why a finding is real, why a proof of concept matters, and why a fix removes or fails to remove the root cause.

A good training path may look different:

- Review agent-generated findings against evidence packages.
- Reproduce selected findings by hand.
- Investigate inconclusive cases.
- Improve test cases and eval examples.
- Write small detection rules.
- Compare model severity against human severity.
- Help convert repeated review lessons into playbooks.

There may also be a new role or responsibility inside security teams: a security pipeline engineer.

That person may not own every finding. Instead, they own the machinery that makes findings trustworthy: sandbox infrastructure, tool schemas, eval harnesses, evidence formats, logging, prompt and model versioning, kill-switch behavior, and regression testing.

In smaller teams, this may be a hat someone wears part time. In larger teams, it may become a real platform role.

Either way, somebody has to own the system.

### Accountability Is the Center

NIST's AI Risk Management Framework is organized around govern, map, measure, and manage. That language is useful here because agentic security workflows are not only technical systems. They are risk systems. Source: [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework).

Governance answers who owns the workflow.

Mapping answers what risks the workflow introduces.

Measurement answers whether the workflow is working.

Management answers what changes when it fails.

The model does not own any of that.

People do.

A security leader should be able to ask:

- What actions can this agent take?
- What evidence does it collect?
- What does it never do without approval?
- How do we know it still works after a model change?
- Who reviews accepted findings?
- Who approves exceptions?
- Who owns a missed issue?

If those questions do not have answers, the organization does not have an AI-native security workflow. It has an AI-assisted experiment.

### Final Thought

LLMs can make security work faster.

They can read more code, connect more context, generate more tests, and draft better reports than a human team could do manually at the same pace.

But speed is not the same as trust.

For serious security work, the model's output has to become evidence. The evidence has to be reproducible. The workflow has to be governed. The human decision has to remain visible.

That is the next level of AI in security.

Not agents that sound confident.

Systems that can prove what they claim.
