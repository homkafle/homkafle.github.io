---
layout: post
title: Unreachable Is a Claim, Not a Conclusion
categories: [Product Security]
tags: [ai, appsec, vulnerability-management, reachability, risk, security-engineering]
fullview: false
description: Why reachability and business context are becoming central to AI-assisted vulnerability triage, and why every unreachable verdict needs evidence, an expiration date, and human ownership.
comments: false
---

### Introduction

Every vulnerability program has a quiet problem.

The queue keeps growing, but not every item in the queue represents the same risk. Some vulnerabilities sit in code paths nobody calls. Some are blocked by network boundaries. Some touch systems with little business impact. Others look boring on paper but sit one hop away from sensitive data.

The hard part is no longer finding more vulnerabilities.

The hard part is knowing which ones are real for your environment today.

That is why reachability and business context matter. Reachability asks whether the vulnerable code, package, service, or configuration can actually be reached. Business context asks what happens if it can.

AI can help with both because the work is full of cross-referencing: code paths, dependency graphs, cloud exposure, identity permissions, data classification, runtime telemetry, compensating controls, and past incidents.

But there is a trap.

When a system says "unreachable," people relax.

That word can be useful. It can also become dangerous if it is treated as a permanent truth. Unreachable is not a conclusion. It is a claim about a specific system, at a specific time, based on specific evidence.

That claim needs an owner.

Scope note: the examples in this post are generalized and hypothetical. They do not describe any employer, client, production system, internal process, or confidential workflow.

### CVE Counting Was Already Failing

AI did not invent the problem of vulnerability prioritization.

Security teams have known for years that raw CVE counts are a weak way to run a remediation program. A list of vulnerabilities does not tell you which ones attackers are using, which ones are exposed in your environment, which ones touch sensitive data, or which ones are blocked by existing controls.

CVSS still matters. FIRST describes CVSS as a way to capture the principal characteristics of a vulnerability and produce a severity score that can help organizations assess and prioritize vulnerability management. CVSS also includes environmental concepts that can be tailored to a deployment. Source: [FIRST CVSS](https://www.first.org/cvss/).

But severity is not the same as likelihood.

That is where EPSS helps. FIRST describes the Exploit Prediction Scoring System as a data-driven model that estimates the probability that a published CVE will be exploited in the wild in the next 30 days. Source: [FIRST EPSS](https://www.first.org/epss/).

CISA's Known Exploited Vulnerabilities catalog and Stakeholder-Specific Vulnerability Categorization are part of the same broad movement: move away from raw vulnerability lists and toward action decisions based on exploitation, exposure, and impact. Source: [CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) and [CISA SSVC](https://www.cisa.gov/stakeholder-specific-vulnerability-categorization-ssvc).

Reachability and business context add another layer.

EPSS, KEV, CVSS, and SSVC help answer broad questions about severity, exploitation, and prioritization. Reachability asks a local question: is this vulnerable thing actually reachable in this environment? Business context asks the next local question: if it is reachable, what does it put at risk?

That local truth is where AI-assisted triage becomes interesting.

It is also where false confidence can hide.

### What Reachability Really Means

Reachability is not one check.

It is a chain of claims.

First, there is exposure.

Can an attacker reach the component through a network path, identity path, user workflow, message queue, API gateway, scheduled job, or internal service call? A critical vulnerability in a public-facing service is not the same as the same vulnerability in a batch process behind several controls.

Second, there is code reachability.

Is the vulnerable function, method, package path, or configuration actually used by the application? Importing a library does not mean every vulnerable function inside it executes. A vulnerability in a parser may not matter if the application never calls that parser on attacker-controlled input.

Third, there is input reachability.

Can untrusted input reach the vulnerable sink? This matters because many vulnerable packages are present but never receive hostile input in the way the advisory describes.

Fourth, there is runtime reachability.

Does the vulnerable code load or execute in the deployed environment? Static analysis can miss dynamic loading, reflection, plugin behavior, feature flags, and configuration-driven paths. Runtime signals can help, but they are still evidence from a particular moment, not proof for all time.

This is especially important for third-party libraries. A package may be present because a framework brings it in transitively. It may be unused in normal traffic. It may also become reachable through an optional module, a deserialization path, a file parser, a plugin, or a configuration change that only appears in one deployment.

A useful reachability verdict should say which of these layers were checked.

"Package present" is not reachability.

"Function loaded" is not full exploitability.

"No observed runtime use" is not the same as "cannot be reached."

"Not seen in this test window" is not the same as "impossible."

The language matters because teams make decisions from it.

### Business Context Decides the Action

Reachability narrows the queue. It does not decide the business response by itself.

Two reachable vulnerabilities can deserve very different treatment.

One might affect a service that processes anonymized telemetry in a low-risk environment. Another might affect a service that has access to customer data, payment flows, deployment credentials, or production secrets.

The difference is business context.

Useful context includes:

- Data sensitivity.
- Public exposure.
- Tenant boundaries.
- Authentication and authorization requirements.
- Service privileges.
- Cloud identity permissions.
- Connection to production systems.
- Existing compensating controls.
- Active exploitation signal.
- Compliance or customer commitments.
- Recovery options.

AI can help gather and connect this context. It can read advisory details, inspect a dependency graph, query cloud configuration, summarize a threat model, check whether a service touches classified data, and draft a risk explanation.

But the conclusion still has to be evidence-backed.

The model can say, "This appears lower priority because the vulnerable package is not reachable from an exposed path and the service does not process sensitive data."

That should not be the end.

The system should show the route map, the call graph edge, the runtime observation, the data-classification record, or the compensating control that supports the claim.

And if that evidence is incomplete, the verdict should say so.

### The Dangerous Word Is Unreachable

False positives are annoying. False "unreachable" verdicts are worse.

A false positive wastes time. A false unreachable label can remove a real vulnerability from the team's attention.

Reachability can be wrong in quiet ways.

A static call graph may miss reflection or dynamic dispatch. A feature flag may be off today and on next sprint. A private service may become reachable after an emergency firewall change. A dependency may be unused in normal traffic but loaded during a rarely used import path. A compensating control may block the example payload but fail against a small variation.

This is why binary labels are often too clean.

Some findings are reachable. Some are not reachable based on current evidence. Some are partially reachable. Some are mitigated. Some are simply inconclusive.

This is why "unreachable" should never be a permanent label.

It should include:

- What was checked.
- What evidence supports the verdict.
- Which code, dependency, container, or cloud state was used.
- When the check happened.
- What changes should trigger revalidation.
- Who accepted the risk of deprioritizing it.

Without that, "unreachable" is just a confident guess with a nice interface.

### AI Helps, But It Needs Source of Truth

The tempting story is that better models will read everything and solve triage.

That is too simple.

AI can help most when it has reliable source-of-truth data to reason over. It needs current asset inventory, dependency metadata, cloud configuration, service ownership, data classification, runtime observations, threat models, and scanner output.

If those inputs are stale, the model will produce a polished answer over stale reality.

The best use of AI here is not replacing the graph, inventory, or telemetry systems. It is making those systems easier to connect and easier for humans to question.

A model can ask better questions:

- Is this vulnerable package actually loaded?
- Which services import it?
- Which endpoints reach the affected function?
- Which identities can call those endpoints?
- Which data stores does the service touch?
- Which compensating controls claim to block exploitation?
- When was the last runtime observation?

Then it can summarize the answer in language humans can act on.

But the evidence should still come from systems that can be inspected and re-run.

### Compensating Controls Need Proof

Compensating controls are useful. They are also easy to overtrust.

A WAF rule may block the proof-of-concept payload but miss encoded variants. A network rule may block public traffic but allow access from a partner network. An EDR policy may be in detect mode, not block mode. A feature flag may disable one path but leave another path open.

So a compensating-control claim needs the same discipline as a vulnerability claim.

It should answer:

- What control is in place?
- What exact behavior does it block?
- Was it tested or only read from configuration?
- What payload or path was used during testing?
- What logs show the block?
- What would make the control stale?
- Who accepted the residual risk?

AI can draft the risk acceptance. It can attach evidence. It can explain the mitigation clearly.

It should not be the signer.

### The Evidence Package

Every reachability decision should leave behind an evidence package.

For a high-risk finding, that package should include:

- Vulnerability identifier.
- Affected asset or service.
- Dependency version or component version.
- Exposure state.
- Code path or call graph evidence.
- Runtime evidence, if available.
- Data classification.
- Service privileges.
- Compensating controls.
- Active exploitation signals such as KEV or EPSS.
- Verdict: reachable, unreachable, partially reachable, mitigated, or inconclusive.
- Timestamp.
- Revalidation triggers.
- Human owner or approver.

The most important part is not the format.

The most important part is that the verdict can be challenged later.

If the network changes, re-check it. If the package version changes, re-check it. If a feature flag changes, re-check it. If a new transitive dependency appears, re-check it. If the service starts touching a new data store, re-check it.

Reachability without revalidation is a snapshot pretending to be a decision.

For compliance-driven work, this matters even more.

If a finding is deprioritized because it is unreachable or mitigated, an auditor or risk owner may later ask who accepted that conclusion. The evidence package should answer that. The model can draft the justification, but a named person or approved governance path has to own the risk decision.

### How to Know It Is Working

This kind of triage system should be measured like a security control.

Useful metrics include:

- Reduction in actionable alert volume.
- False unreachable rate from labeled test cases or later human review.
- Percentage of unreachable findings that later reopen.
- Time between environment change and revalidation.
- Percentage of compensating-control claims that are actively tested.
- Percentage of risk acceptances with human sign-off.
- Mean time from vulnerability discovery to triage decision.
- Human override rate.
- Percentage of verdicts marked inconclusive.
- Developer trust in the prioritized queue.
- Number of high-risk findings found outside the triage process.

The last metric matters.

If engineers keep manually re-triaging because they do not trust the reachability system, the system has not earned its place. If major issues keep appearing outside the prioritized queue, the system is missing reality.

The goal is not to make the dashboard look clean.

The goal is to help teams act on the right risk sooner.

### The Skill This Asks For

This is not just an LLM prompting skill.

It is security engineering, dependency analysis, cloud literacy, and data modeling at the same time.

The people building these systems need to understand:

- Call graphs and taint analysis.
- Dependency and transitive dependency behavior.
- Cloud network exposure.
- IAM and service identity.
- Data classification.
- Runtime observability.
- Compensating-control testing.
- Evaluation sets for reachability accuracy.
- Revalidation triggers.
- Human risk ownership.

That is a lot.

But it is the work underneath the promise of AI-assisted triage.

Without it, AI will only make vulnerability reports sound more confident.

With it, AI can help teams make better decisions faster.

### Final Thought

Reachability and business context are not new ideas.

What is changing is the cost of doing them repeatedly.

AI can help connect advisory details, dependency graphs, code paths, cloud configuration, runtime evidence, and business context. That is a real gain. It can reduce noise and help teams focus on vulnerabilities that matter in their environment.

But the discipline does not change.

An unreachable verdict is a security claim.

It needs evidence. It needs a timestamp. It needs a revalidation trigger. It needs a human owner.

Otherwise, it is just another way for risk to hide.
