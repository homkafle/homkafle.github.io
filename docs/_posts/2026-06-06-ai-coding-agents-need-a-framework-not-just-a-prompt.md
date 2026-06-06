---
layout: post
title: AI Coding Agents Need a Framework, Not Just a Prompt
categories: [Artificial Intelligence]
tags: [ai-agents, appsec, claude-code, codex, mcp, security-engineering]
fullview: false
description: Lessons from building a reusable framework for AI coding agents, with examples from application security review and vulnerability validation.
comments: false
---

### Introduction

I use AI coding agents heavily in my daily work. Most of my hands-on work has been with Claude Code CLI, usually with Opus and Sonnet models, but the lesson I keep coming back to is not tied to one provider.

The lesson is simple: agents perform better when they inherit a working environment, not just a prompt.

A good prompt can help for one task. A good framework helps every task after that. By framework, I mean a structured folder or workspace that gives the agent the things it needs to work well: instructions, skills, hooks, MCP servers, permission rules, commands, examples, and known file paths.

This has become part of my workflow. I run the agent, observe where it does well or poorly, collect feedback, and then improve the framework with another agent that understands the framework's purpose. Over time, the framework gets better. The prompts get shorter. The agent makes fewer wrong assumptions.

That is the real value. The model is powerful, but the system around the model is what makes the work repeatable.

### Why Prompts Are Not Enough

A prompt is temporary. It lives in one conversation. It may be carefully written, but it is easy to forget, hard to audit, and usually disconnected from the workspace.

A framework is different. It gives the agent a stable way to learn how the work should be done.

For example, a prompt might say:

> Review this code for security issues.

A framework can say:

- The threat model lives in `docs/security/threat-model.md`.
- API authorization rules live in `docs/security/authz-rules.md`.
- Use the secure code review checklist in `skills/appsec-review/SKILL.md`.
- Treat scanner output as untrusted until validated.
- Do not run exploit code against production hosts.
- Ask for approval before using tools that send data outside the workspace.
- After review, write findings with impact, exploitability, affected path, and remediation.

That is a different level of guidance. The agent no longer has to guess how the team thinks about security. It has a map.

### Best Practice 1: Treat Instructions as Infrastructure

Instructions should be versioned, reviewed, and improved like code.

Different tools use different names. Claude Code uses files such as `CLAUDE.md` for persistent project instructions. OpenAI Codex can use `AGENTS.md` to guide how it works in a repository. Other tools have their own versions of the same idea. The provider does not matter as much as the pattern.

Keep the main instruction file short. Put only the things the agent should remember every time.

Good always-on instructions include:

- Project layout.
- Build and test commands.
- Coding standards.
- Security rules.
- Approval expectations.
- What not to touch.
- Where to find deeper guides.

Avoid turning the main instruction file into a book. If everything is always loaded, the agent has more to carry and more chances to miss what matters. A good rule is to keep the main file focused and link to deeper guides when needed.

Example:

```md
# Agent Instructions

- Read `docs/security/review-guide.md` before doing application security review.
- Run `npm test` before committing frontend changes.
- Never change files under `infra/prod/` without explicit approval.
- Treat SAST output as a lead, not as proof.
- For security findings, include impact, exploitability, evidence, and remediation.
```

This is simple, but it changes the agent's behavior.

### Best Practice 2: Separate General Rules from Skills

Not every instruction belongs in the always-on guide.

Some instructions are task-specific. A secure code review workflow does not need to load during a CSS cleanup. A release checklist does not need to load during threat modeling. A scanner triage guide does not need to load while writing documentation.

That is where skills, playbooks, or task guides help.

A security framework might include:

- `skills/appsec-review/SKILL.md`
- `skills/sast-triage/SKILL.md`
- `skills/exploit-validation/SKILL.md`
- `skills/threat-modeling/SKILL.md`
- `skills/secure-fix-review/SKILL.md`

Each skill should be small and practical. It should tell the agent what to inspect, what evidence to collect, and what output format to use.

Example skill for reviewing API code:

```md
# AppSec API Review

Use this when reviewing backend API changes.

Check:
- Authentication is required where expected.
- Authorization is enforced on object access.
- User-controlled input is validated before use.
- Sensitive data is not logged.
- Rate limits exist for expensive or abuse-prone actions.
- Errors do not expose secrets or internal details.

Output:
- Finding title
- Affected file and function
- Why it matters
- How it could be abused
- Suggested fix
```

This helps the model stay focused. It also helps humans review the model's work because the output is consistent.

### Best Practice 3: Use Hooks for Guardrails

Prompts guide behavior. Hooks can enforce parts of the workflow.

Hooks are useful for actions that should happen the same way every time. They can run before or after tool use, block risky commands, format files, scan for secrets, or remind the agent about a policy at the right moment.

For security work, hooks can be very useful.

Examples:

- Block commands that target production domains.
- Warn before running exploit scripts.
- Run a secrets scan before commit.
- Run formatting after file edits.
- Block edits to generated files unless explicitly allowed.
- Log which MCP tools were called during a review.

A hook does not replace human judgment, but it removes some easy mistakes from the workflow.

Example:

```text
Before running shell commands:
- Block commands containing production hostnames.
- Ask for approval before network scanning.
- Ask for approval before running files under `poc/` or `exploits/`.
```

This matters because agents can move quickly. Speed is useful, but only when the boundaries are clear.

### Best Practice 4: Treat MCP Servers Like Real Integrations

MCP gives agents a standard way to connect with tools and data sources. That can be powerful. It can also be risky.

An MCP server might expose access to tickets, code search, cloud assets, logs, documentation, scanners, or internal services. That means it should be treated like any other integration with sensitive access.

Good practices include:

- Expose the smallest useful tool surface.
- Use read-only access when write access is not needed.
- Keep credentials scoped and rotated.
- Log tool calls.
- Validate tool inputs.
- Avoid sending sensitive data to tools that do not need it.
- Document what each MCP server is allowed to do.

For a security engineer, this can be the difference between a helpful agent and an overpowered one.

Example:

Suppose a scanner reports possible SQL injection in `GET /api/orders?sort=created_at`. The agent should be able to read the scanner finding, inspect the route, inspect the query builder, and review related tests. It probably does not need access to production data or the ability to run live payloads against a real customer environment.

The framework should make that clear.

### Best Practice 5: Match the Model to the Work

Not every task needs the strongest model.

In my experience, stronger reasoning models are best for unclear work. Use them when the task needs judgment, tradeoff analysis, or deep code understanding.

Good tasks for stronger models:

- Planning a security review.
- Understanding a complex authorization flow.
- Validating whether a SAST finding is exploitable.
- Reviewing a proposed fix for bypasses.
- Writing a threat model.
- Deciding whether a vulnerability is reachable.

Faster or cheaper models are often enough for more mechanical work.

Good tasks for smaller or faster models:

- Formatting output.
- Summarizing scanner findings.
- Updating a checklist.
- Applying a known pattern.
- Drafting test cases from a confirmed issue.
- Cleaning documentation.

The framework should support both. The same folder structure, skill files, examples, and output formats should help any capable model work in the same direction.

### Example 1: Reviewing Application Code for Vulnerabilities

Imagine a security engineer is asked to review a pull request that adds a new API endpoint:

```text
POST /api/projects/{projectId}/invite
```

The endpoint invites a user to a project. At first glance, the code looks normal. It checks that the caller is logged in. It validates the email address. It sends an invite.

But the real security question is not "does this code run?" The question is "who is allowed to invite users to this project?"

Without a framework, the agent may only check for obvious issues like input validation or error handling.

With a framework, the agent can be guided to:

- Read the authorization rules.
- Check whether project membership is verified.
- Check whether the caller must be an owner or admin.
- Look for object-level authorization.
- Check whether invitations can escalate privilege.
- Review logs for sensitive invite tokens.
- Suggest tests for unauthorized invitation attempts.

The agent might produce a finding like this:

```text
Finding: Missing object-level authorization on project invite

The endpoint verifies that the caller is authenticated, but it does not verify
that the caller has admin rights for the target project. A regular user who can
guess or obtain a project ID may be able to invite users into a project they do
not control.

Suggested fix:
- Load the caller's role for the target project.
- Require owner or admin role before creating the invite.
- Add tests for non-member, member, admin, and owner cases.
```

That is useful. It is not useful because the model is magical. It is useful because the framework pointed the model toward the right question.

### Example 2: Validating a SAST Finding

Now imagine a SAST tool reports possible command injection.

The scanner points to code like this:

```text
run("convert " + filename + " " + outputPath)
```

A weak workflow stops at the scanner result. A stronger workflow asks whether the finding is real, reachable, and exploitable.

The framework can guide the agent to check:

- Where `filename` comes from.
- Whether the value is user-controlled.
- Whether validation is allowlist-based.
- Whether shell execution is used.
- Whether arguments can be passed as an array instead of a string.
- Whether tests cover shell metacharacters.
- Whether the vulnerable path is exposed to normal users.

The agent may find that `filename` comes from an uploaded file name and is later passed into a shell command. It may also find that the application strips path traversal but does not remove shell metacharacters.

The final output should not simply say "command injection." It should explain the chain.

Example:

```text
Finding: User-controlled filename reaches shell command

The scanner finding appears valid. `filename` is derived from the uploaded file
name. The code removes path separators, but it still builds a shell command with
string concatenation. An attacker may be able to inject shell metacharacters if
the upload path is reachable by an authenticated user.

Suggested fix:
- Avoid shell string construction.
- Pass arguments as an array to a safe process execution API.
- Generate server-side file names instead of trusting uploaded names.
- Add tests for shell metacharacters such as `;`, `&`, `|`, and backticks.
```

This is where AI can help a lot. It can trace code, connect files, propose tests, and draft a fix. But the human still needs to decide whether the path is reachable, whether the impact is real, and whether the proposed fix is safe.

### Best Practice 6: Make Feedback Part of the System

The framework should improve after every serious use.

When the agent makes a mistake, do not only correct the conversation. Ask whether the framework should change.

Good feedback to capture:

- The agent missed an important file.
- The agent used the wrong test command.
- The agent trusted scanner output too quickly.
- The agent forgot the expected report format.
- The agent tried to use a tool that should be gated.
- The agent kept asking for context that could be documented.
- The agent misunderstood the business rule.

This is where the framework becomes a living asset.

Example:

If the agent keeps missing object-level authorization, add a rule to the appsec review skill:

```md
For every endpoint that accepts an object ID, check whether the caller is
authorized to act on that specific object. Authentication alone is not enough.
```

That one line may prevent the same mistake in future reviews.

### Potential Problems

This workflow is powerful, but it has risks.

The first risk is stale context. If the framework says tests run with `npm test`, but the repo moved to `pnpm test`, the agent will keep doing the wrong thing with confidence.

The second risk is too much context. If every guide loads all the time, the agent may become less focused. Keep global instructions small and move detailed workflows into skills or scoped guides.

The third risk is over-permissioning. An agent that can read tickets, query scanners, access logs, run shell commands, and call internal tools may become too powerful for routine work. Permissions should match the task.

The fourth risk is false confidence. AI can produce clean explanations for findings that are not real. For security work, every important claim still needs evidence.

The fifth risk is framework drift. If the framework is not maintained, it slowly becomes a museum of old assumptions.

### Future Opportunities

I think this space is still early.

The next step is not just better prompts. It is better agent operations.

Useful future ideas include:

- Framework linting that checks broken paths, stale commands, and conflicting rules.
- Regression tests for agent workflows using known tasks.
- Security review templates that require exploitability, impact, and evidence.
- MCP permission profiles for different task types.
- Run logs that show which skills, tools, and files helped the agent.
- Framework review agents that suggest updates after each task.
- Provider-neutral layouts that work across Claude, Codex, and other coding agents.

The long-term opportunity is to make agent work more repeatable, observable, and safe.

### Closing Thoughts

The best agent workflow I have found is not a single prompt. It is a loop.

Give the agent a clear framework. Let it work. Watch where it struggles. Improve the framework. Repeat.

For security engineering, this matters even more. Reviewing code, validating scanner findings, and testing exploitability all require context. The model can help move faster, but the framework helps it move in the right direction.

The model is replaceable.

The workflow is the asset.

And the better the framework gets, the better every future agent run becomes.

### References

- [Claude Code memory and instructions](https://docs.anthropic.com/en/docs/claude-code/memory)
- [Claude Code hooks](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [Claude Code settings](https://docs.anthropic.com/en/docs/claude-code/settings)
- [Model Context Protocol security best practices](https://modelcontextprotocol.io/specification/2025-06-18/basic/security_best_practices)
- [OpenAI Codex CLI getting started](https://help.openai.com/en/articles/11096431)
