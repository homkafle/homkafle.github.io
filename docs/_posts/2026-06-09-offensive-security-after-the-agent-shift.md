---
layout: post
title: Offensive Security After the Agent Shift
categories: [Product Security]
tags: [ai, appsec, offensive-security, claude, agents, mcp, security-engineering]
fullview: false
description: How frontier AI models are changing offensive security work, what an AI-native security operation may look like, and what human skills will still matter.
comments: false
---

### Introduction

For a long time, application security work followed a familiar rhythm.

A team would threat model a system, run static analysis, review code by hand, test the running application, validate findings, review fixes, and eventually decide whether the application was ready for production. If a patch came back, the work started again: did the fix address the root cause, did it introduce a bypass, and did it reduce risk in the real system?

That work still matters. But the operating model around it is changing fast.

Frontier models like Claude Opus, Claude Fable, Claude Mythos, OpenAI's reasoning models, and the agent systems built around them are starting to absorb large parts of the traditional security review loop. They can read code, follow data flow, inspect configuration, generate tests, reason across documentation, and run tools. They can also be wired into workflows with skills, MCP servers, hooks, permission rules, and review gates.

This is not just better autocomplete. It is the beginning of a new security production system.

The important question is no longer whether AI can help a security engineer. It can. The better question is: what happens to offensive security when AI becomes a first layer of analysis, execution, and verification?

I do not think the answer is "humans disappear." I think the answer is more interesting. The work becomes more leveraged, more continuous, and more strategic. The best security engineers become designers of security systems, not just reviewers of individual findings.

### The Old AppSec Assembly Line

Many AppSec programs were built around queues.

Code review queues. Threat model queues. Scanner queues. Penetration test queues. Exception queues. Patch review queues. Production approval queues.

This made sense when expertise was scarce and manual review was the main way to reason deeply about a system. A skilled security engineer could look at a pull request and notice the missing authorization check that a scanner would miss. They could see that a logging change created a privacy issue. They could recognize that a "low severity" bug became serious because it crossed a trust boundary.

The best teams did not just find bugs. They made risk calls. They answered hard questions:

- Is this reachable?
- Is this exploitable in our environment?
- Does this cross a meaningful trust boundary?
- Can this be chained with another weakness?
- Does the fix remove the class of issue or only quiet the symptom?
- Is the product safe enough to ship?

That kind of work requires judgment. It also takes time.

The problem is that software delivery got faster than the old review model. Cloud services, microservices, third-party packages, AI-generated code, infrastructure-as-code, and continuous deployment all increased the surface area. Security teams were asked to be deep, fast, and everywhere at once.

That was already unsustainable before frontier AI. Now the pressure is even higher because models can also help attackers move faster.

### What Frontier Models Have Changed

The most important shift is that models are no longer limited to answering questions. They are becoming task runners.

Anthropic described Claude Opus 4 as built for coding, advanced reasoning, and AI agents, with Claude Code becoming generally available and support for tools, GitHub workflows, IDE integrations, and API capabilities such as code execution and MCP connectors. In August 2025, Anthropic said Claude Opus 4.1 improved agentic tasks, real-world coding, and reasoning, and reported 74.5% on SWE-bench Verified. Those are not security benchmarks, but they matter because much of modern security work is deep software understanding. Sources: [Introducing Claude 4](https://www.anthropic.com/news/claude-4) and [Claude Opus 4.1](https://www.anthropic.com/news/claude-opus-4-1).

The bigger signal arrived with Anthropic's June 9, 2026 launch of Claude Fable 5 and Claude Mythos 5. Anthropic described Fable 5 as a "Mythos-class" model for general use with safeguards, and Mythos 5 as the same underlying model with some safeguards lifted for selected cyberdefenders and infrastructure providers. Anthropic also said Mythos 5 has the strongest cybersecurity capabilities of any model in the world. That is Anthropic's claim, but even if we discount the marketing, the direction is clear: the highest-capability models are being treated as dual-use security infrastructure, not just productivity tools. Source: [Claude Fable 5 and Claude Mythos 5](https://www.anthropic.com/news/claude-fable-5-mythos-5).

At the same time, the tooling around models is getting more mature. Claude Code supports skills, hooks, and MCP connections. Skills turn repeatable workflows into reusable instructions. Hooks can run at lifecycle points such as before tool use, after tool use, or when instructions are loaded. MCP lets the model connect to external systems such as issue trackers, databases, APIs, logs, and code hosts. OpenAI also documents MCP as an open protocol for extending models with tools and knowledge. Sources: [Claude Code skills](https://code.claude.com/docs/en/skills), [Claude Code hooks](https://code.claude.com/docs/en/hooks), [Claude Code MCP](https://code.claude.com/docs/en/mcp), and [OpenAI MCP docs](https://developers.openai.com/api/docs/mcp).

Put those pieces together and the security workflow changes.

The model is no longer a side window where an engineer pastes a function and asks, "Do you see a bug?" The model becomes an agent inside the engineering environment. It can read the pull request, inspect the service, consult the threat model, run tests, compare scanner output, write a proof-of-concept in a safe local harness, propose a fix, and explain what evidence supports the conclusion.

This is a different game.

### The New Offensive Security Role

Offensive security has often been defined by the ability to find and exploit weaknesses. That will still matter. But in an AI-native operation, the role expands.

The future offensive security engineer will spend less time manually walking the same checklist and more time designing the system that walks it every day.

The job becomes:

- Define the attack surface.
- Encode the review methodology.
- Build skills for recurring security tasks.
- Connect the right tools through MCP with least privilege.
- Create hooks that block unsafe actions and enforce evidence collection.
- Review the model's reasoning and challenge its assumptions.
- Convert validated findings into durable fixes.
- Teach the organization what the findings mean.

This is closer to being a security operator, system designer, and adversarial product thinker at the same time.

The strongest engineers will not be the people who simply ask the model for "security issues." They will be the people who can build a governed workflow where the model knows what to inspect, what evidence to collect, what it is allowed to do, and when a human must decide.

### From Manual Review to Agentic Review

Imagine a pull request that adds a new endpoint:

```text
POST /api/projects/{projectId}/invite
```

In the old model, a security engineer might manually read the controller, service layer, authorization checks, validation logic, logs, tests, and related documentation. They might run the application and test a few role combinations.

In an AI-native workflow, an agent can do the first pass:

- Read the diff and identify the changed trust boundaries.
- Pull the product authorization rules from a security guide.
- Search for similar endpoints.
- Compare the implementation against expected role behavior.
- Generate tests for owner, admin, member, and outsider roles.
- Run the tests locally.
- Check whether email addresses or invite tokens are logged.
- Review whether rate limiting exists.
- Produce a finding only if there is evidence.

The human engineer then reviews the agent's work. Not as a passive approver, but as the person responsible for reality.

Did the agent understand the product model? Did it test the right roles? Did it miss a business rule? Did it overstate impact? Is the proposed fix too broad? Could the patch break a legitimate workflow?

This is where human judgment remains central. AI can accelerate the search. It cannot own the consequences.

### The AI-Native Security Operating Model

I think mature organizations will move from "security team reviews everything" to "security system continuously reasons about risk."

That system will have several layers.

First, there will be a source-of-truth layer. Threat models, authorization rules, secure coding standards, data classification, exception decisions, and service ownership need to exist in a form that agents can read. If the only source of truth is tribal knowledge, the model will guess.

Second, there will be a tool layer. Agents will connect to code search, CI, SAST, DAST, SCA, cloud inventory, tickets, logs, design docs, and local test environments. MCP and similar standards matter because they make these integrations repeatable.

Third, there will be a governance layer. Hooks, permissions, model routing, audit logs, and approval gates will decide what the agent can do. A review agent may read production logs in aggregate, but not fetch customer records. It may generate a local exploit test, but not run payloads against production. It may open a pull request, but not merge without review.

Fourth, there will be a verification layer. Findings will need evidence. Fixes will need tests. Agent output will need to be reproducible. The best organizations will treat model conclusions the way good security teams already treat scanner output: useful leads, not automatic truth.

Finally, there will be a learning layer. Every false positive, missed issue, weak patch, and strong detection becomes feedback. The organization improves its skills, prompts, rules, examples, and test harnesses. The system gets better because the humans teach it how the organization thinks.

This is how AI-native operations will differ from simple automation. Automation runs a task. An AI-native security operation learns how the work should be done.

### How the Workforce Changes

The security workforce will not be smaller in the way many people imagine. It will be reshaped.

Some repetitive work will shrink. Teams will spend less time manually summarizing scanner output, copying evidence between tools, writing boilerplate tests, or reviewing the same pattern for the hundredth time.

But new work will grow.

We will need security engineers who can build and govern agents. We will need people who understand model behavior, tool permissions, prompt injection, data leakage, and agent failure modes. We will need engineers who can turn an expert workflow into a skill file, a test harness, and a review rubric.

We will need security platform teams that treat AI agents as part of the internal developer platform. We will need model risk reviewers who understand both security and AI behavior. We will need incident responders who can investigate not only what a human did, but what an agent did with a toolchain.

The entry-level path will also change. Junior engineers may start by supervising agents, validating findings, improving playbooks, and learning from the model's work. That can be powerful if done well. It can also be dangerous if organizations let people skip the fundamentals.

We should not train a generation of engineers who only know how to ask an agent. We should train engineers who know enough to argue with one.

### The Attacker Side Is Moving Too

This shift is not only defensive.

Anthropic's June 2026 analysis of 832 banned accounts used for malicious cyber activity found that threat actors were using AI deeper in the attack lifecycle, including later-stage activity such as account discovery and lateral movement. Anthropic also argued that higher-risk actors are building scaffolding around models to chain attack stages with less human input. Source: [What we learned mapping a year's worth of AI-enabled cyber threats](https://www.anthropic.com/news/AI-enabled-cyber-threats-mitre-attack).

That is the uncomfortable part. The same agentic patterns that help defenders can help attackers.

An attacker can also create skills. An attacker can also connect tools. An attacker can also build a loop that reads output, chooses the next step, and keeps going.

This means defenders cannot rely on old signals alone. A low-skill actor with a strong agent may behave like a higher-skill actor. A simple interface may hide a complex workflow. A small number of prompts may trigger a long chain of tool use.

Security teams will need to detect orchestration, not just payloads. They will need to watch for tool chains, abnormal decision loops, unusual automation paths, and AI-assisted reconnaissance. The unit of analysis becomes the workflow.

### What Will Not Change

The models will get better. Context windows will get larger. Tool use will become smoother. Agents will become cheaper and more persistent.

But some human responsibilities will not go away.

Ethics will not go away. Offensive security is powerful because it studies harm before harm happens. That power needs restraint. Just because an agent can find or chain an exploit does not mean every test should be run everywhere.

Accountability will not go away. A model cannot be accountable to a customer, a regulator, a patient, a citizen, or a business owner. People and organizations remain responsible for what they deploy.

Context will not go away. A model can read a policy, but a human often understands why the policy exists. It may know that a feature is tied to a legal promise, a customer commitment, a safety requirement, or a past incident that changed how the organization thinks.

Taste will not go away. Good security engineering has taste. It knows when a fix is simple and durable, when a control is too brittle, when a process creates theater instead of safety, and when a finding needs a better story to drive action.

Courage will not go away. Security often requires telling people what they do not want to hear. A release may need to pause. A design may need to change. A shortcut may need to be challenged. Models can support that work, but humans still have to carry the decision.

Care will not go away. The goal is not to win arguments or produce more tickets. The goal is to protect people, data, trust, and the systems that modern life depends on.

### A More Prosperous Security Future

There is a hopeful version of this future.

In that version, AI gives small teams the kind of security leverage that only large organizations used to have. Open-source maintainers get help finding dangerous bugs before attackers do. Startups get secure design review without waiting weeks. Hospitals, schools, local governments, and small businesses get stronger defenses. Security knowledge becomes more available.

The path to that future is not blind acceleration. It is governed acceleration.

We should embrace frontier models, but we should do it with clear boundaries:

- Use AI to reduce toil, not to remove responsibility.
- Use agents to make evidence easier to collect, not easier to fake.
- Give models the least privilege needed for the task.
- Keep humans in the loop for high-impact decisions.
- Treat security knowledge as infrastructure.
- Share defensive lessons widely.
- Measure outcomes, not activity.

If we do this well, offensive security becomes less about heroic last-minute review and more about continuous immune response. The system sees more, tests more, learns more, and helps humans make better decisions.

### Final Thought

The old security model asked a small group of experts to manually stand between software velocity and production risk.

The new model asks those experts to build the machines, workflows, and judgment systems that make security scale.

That is a bigger job, not a smaller one.

Claude Mythos, Opus, Fable, and the frontier models that follow are changing what is possible. They can find bugs faster. They can review patches faster. They can run more tests, read more code, and connect more tools than a human team could do manually.

But the future of security will not be decided by model capability alone.

It will be decided by whether we use that capability with wisdom.

The best offensive security engineers of the next decade will not only know how to break systems. They will know how to build AI-native security operations that make systems harder to break in the first place.
