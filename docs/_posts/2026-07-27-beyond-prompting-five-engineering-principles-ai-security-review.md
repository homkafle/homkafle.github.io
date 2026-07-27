---
layout: post
title: Beyond Prompting: Five Engineering Principles for AI-Assisted Security Review
categories: [Product Security]
tags: [ai, appsec, offensive-security, security-review, ai-agents, security-engineering]
fullview: false
description: Lessons from using AI in offensive security validation, and why evidence, experiments, validation, and reusable systems matter more than clever prompts.
comments: false
---

### Introduction

AI has become one of the tools I use often when thinking through offensive security validation.

In practice, that might mean using it to understand an unfamiliar code path, reason about a vulnerability report, build a small local example, review a proposed fix pattern, or challenge a conclusion before writing it down.

Like many engineers, I started by focusing on prompts.

I experimented with better questions, longer context, stricter instructions, and cleaner output formats. That helped. Better prompts usually produced better answers.

But after a while, I noticed something important.

The biggest improvements were no longer coming from better prompts.

They were coming from improving everything around the prompt.

The way I organized context. The way I structured investigations. The way I separated claims from evidence. The way I validated conclusions. The way I captured lessons from previous reviews.

Over time, I stopped treating AI as a tool that simply answers questions. I started treating it as one participant in an engineering process that I am still learning how to design and govern.

I do not have that process fully figured out. It changes almost every week. New models, new tools, and new mistakes keep challenging assumptions I thought were solid.

Still, five principles have consistently improved the quality of my AI-assisted security work.

Scope note: the examples in this post are generalized. They do not describe any employer, client, production system, internal workflow, or confidential process. Any exploitability or proof-of-concept discussion assumes authorized testing in a controlled environment.

### 1. Evidence Should Always Win

One of the easiest mistakes, whether made by a person or a model, is confusing a convincing explanation with a verified conclusion.

LLMs are very good at producing explanations that sound reasonable. That does not automatically make them correct.

In security review, I try to stop asking only whether I agree with a conclusion. I ask a more useful question:

What evidence supports this claim?

If a security note says a vulnerability is exploitable, I want to know exactly why.

Can the claim be traced to source code? A runtime observation? A configuration? An IAM permission? A request flow? A policy? A reproducible local experiment?

If I cannot trace an important statement back to evidence, it does not matter how well written it is.

For example, an AI review may say:

> This endpoint may allow privilege escalation.

That sentence is not enough.

A useful review should show the endpoint, the identity, the authorization check, the object boundary, and the lower-privilege actor that can reach the higher-privilege behavior.

The same applies to remediation. If the model says "apply least privilege," I want to know which permission should be removed, from which role, and what test or review step confirms that the application still works without it.

Confidence should follow evidence, not the other way around.

### 2. Separate Generation From Validation

One workflow change has probably improved my AI-assisted reviews more than any individual prompt.

I separate generation from validation.

In the generation step, the model can be creative. It can enumerate possible risks, compare patterns, propose abuse cases, inspect code paths, and draft findings.

In the validation step, the model is not asked to defend its own answer. Its job is to challenge the reasoning.

Earlier in my workflow, I would review the model's final report myself and move forward if it looked coherent.

Over time, I noticed something uncomfortable.

The reports were usually coherent. They were usually logical. Sometimes they were also wrong.

Not dramatically wrong. Subtly wrong.

A conclusion might rely on an assumption that had never been tested. A statement might sound stronger than the evidence supported. A finding might reference code without proving that the code was reachable in practice.

So I added a separate review pass. It asks questions like:

- Which claims are directly supported by evidence?
- Which claims rely on inference?
- Can every important statement be traced to code, configuration, policy, or an experiment?
- What assumptions would need to be false for this finding to collapse?
- What evidence would weaken the conclusion?
- What high-risk boundary did the review spend the least time on?

This one change reduced unsupported claims in my reports.

It also reminded me of something that existed long before AI: independent review usually produces better engineering outcomes.

The model can help generate the first pass.

The second pass should behave like a skeptical reviewer.

### 3. Prefer Experiments Over Arguments

Security discussions can easily become debates.

Someone believes a vulnerability is exploitable. Someone else believes it is not. AI can generate arguments for either side.

Arguments rarely settle technical uncertainty.

Experiments do.

Whenever possible, I ask:

What is the smallest safe experiment that would settle this question?

Instead of debating whether an authorization bypass exists, build a test that tries to access another user's object. Instead of arguing about whether an SSRF path is reachable, build a controlled local test that proves whether user input reaches a server-side request. Instead of assuming a proposed fix works, design a test that would fail before the fix and pass after it.

AI is often useful here. It can suggest test cases I had not considered. It can draft local harnesses. It can generate request variants. It can help compare behavior before and after a patch.

But the experiment is what builds confidence.

A good proof of concept is not theater. It is a measurement tool.

For security engineers, this distinction matters. We should not use AI to create exploit code for systems we do not own or have permission to test. But in authorized environments, AI can help us create controlled experiments that replace debate with evidence.

The point is not to prove that we were right.

The point is to discover what is true.

### 4. Preserve Learning Instead of Repeating It

One thing surprised me after enough AI-assisted investigations.

I kept solving the same problems.

The same validation steps. The same missing context. The same useful questions. The same model mistakes. The same reminders about reachability, privilege boundaries, and evidence.

At first, I relied on memory.

That did not scale.

So I started treating every investigation as a chance to improve the workflow itself.

If I discover a useful validation step, it can become part of a checklist. If an instruction consistently improves results, it can become a reusable skill. If a particular experiment keeps proving valuable, it can become part of the standard review process. If a model repeatedly misses a class of issue, I can add a review prompt or evidence requirement for that category.

This matters in security because the same classes of mistakes return again and again: missing object authorization, weak tenant isolation, token forwarding, unsafe logging, overbroad permissions, incomplete fixes, and controls that work in one path but fail in another.

This is where AI-assisted security starts to feel less like a chat session and more like engineering.

A review should not only answer today's question.

It should make tomorrow's review better.

More recently, the idea of context engineering has pushed me to think harder about what the model receives before an investigation starts. The model's output depends heavily on the quality of the context: architecture notes, threat models, authorization rules, previous lessons, safe commands, known false positives, and expected evidence format.

Context engineering is not magic. It is just another way of saying that the system around the model matters.

Small improvements accumulate.

Eventually, the workflow begins carrying experience forward instead of asking one engineer to remember everything.

### 5. Optimize the System, Not the Session

This may be the biggest lesson.

It is easy to judge an AI interaction by asking:

Did I get a good answer?

I think a better question is:

Did this investigation improve how the next investigation will be performed?

Sometimes the improvement is a better prompt.

More often, it is something else:

- A clearer checklist.
- A safer tool boundary.
- A reusable script.
- A better evidence requirement.
- A cleaner context file.
- A stronger claim-evidence table.
- A validation step that future reviews inherit automatically.
- A report format that makes assumptions visible.

Those improvements continue paying dividends regardless of which model I use.

Today a workflow may involve Claude. Tomorrow it may involve Codex. A year from now, it will probably involve something neither of us has used yet.

The models will change.

Good engineering principles tend to last longer.

That is why I care less about one perfect prompt and more about the system around the prompt.

### What This Means for Security Engineers

I think upcoming security engineers should learn AI deeply, but not passively.

Do not only learn how to ask a model for vulnerabilities.

Learn how to structure a review so the model has the right context. Learn how to separate claims from evidence. Learn how to test exploitability. Learn how to challenge remediation. Learn how to turn repeated lessons into reusable workflows.

The important skill is not prompting alone.

The important skill is judgment with leverage.

AI can help us inspect more code, ask more questions, generate more tests, and write clearer reports. But the human engineer still has to decide whether the model understood the system, whether the evidence supports the claim, whether the exploit path is realistic, and whether the fix actually reduces risk.

That responsibility does not go away.

If anything, it becomes more important.

### Final Thought

I am still early in this journey.

If I read this post a year from now, I expect there will be ideas I would change and blind spots I will recognize.

Honestly, I hope that is true.

The pace of AI improvement is extraordinary, and no workflow should be treated as finished.

But one belief has become stronger over time.

The engineers who benefit most from AI will not necessarily be the ones who write the cleverest prompts.

They will be the ones who build trustworthy systems around AI.

Systems that value evidence over confidence.

Systems that separate generation from validation.

Systems that resolve uncertainty through experiments.

Systems that preserve what they learn.

Systems that keep improving long after a single conversation with a model has ended.

That is the direction I am trying to move in.
