---
layout: post
title: "The 87-to-13 Collapse Is a Harness Bug, Not a Model Problem"
categories: [Artificial Intelligence, Security]
tags: [ai-security, llm-security, agentic-ai, collapse, harness, model, problem]
fullview: false
description: "Eighty-seven percent. Thirteen percent. Everyone building an LLM-backed pentest harness has run into this pair of numbers used as proof that lab-condition agents collapse the moment they touch a real…"
comments: false
---

Eighty-seven percent. Thirteen percent. Everyone building an LLM-backed pentest harness has run into this pair of numbers used as proof that lab-condition agents collapse the moment they touch a real target. Here's what nobody says loudly enough: these are two different experiments, run by two different teams, against two different tasks [\[1\]](https://www.ibm.com/think/insights/chatgpt-4-exploits-87-percent-one-day-vulnerabilities)[\[2\]](https://arxiv.org/html/2503.17332v4)[\[3\]](https://arxiv.org/abs/2503.17332). Fang et al. handed GPT-4 the CVE advisory text and watched it exploit 87% of one-day vulnerabilities [\[1\]](https://www.ibm.com/think/insights/chatgpt-4-exploits-87-percent-one-day-vulnerabilities). CVE-Bench, a later and unrelated benchmark, dropped the best available agent framework into forty real web-app CVEs with no advisory hand-holding and got 12.5% [\[2\]](https://arxiv.org/html/2503.17332v4). Nobody ran both conditions against the same models on the same targets. The "87-to-13 collapse" is a tertiary blog's mashup of two papers that don't even cite each other [\[3\]](https://arxiv.org/abs/2503.17332)[\[4\]](https://appsecsanta.com/research/ai-pentesting-agents-2026). That doesn't mean the underlying gap is fake. PACEbench ran seven different models against WAF-defended targets and every single one scored zero [\[5\]](https://arxiv.org/html/2510.11688v1). It means the popular story about what caused the gap has been garbled from the start, and if you're designing a harness around that garbled story, you're solving the wrong problem.

## What the numbers actually measure

Start with Fang et al. It isn't a lab-vs-real experiment on its own — it's a single-condition demonstration with a built-in control. Strip the advisory text out, and GPT-4's success rate on the identical one-day CVEs falls from 87% to 7% [\[6\]](https://www.ibm.com/think/insights/chatgpt-4-exploits-87-percent-one-day-vulnerabilities). That eighty-percent internal collapse already tells you information availability, not raw model intelligence, was doing most of the work in that paper — before anyone introduced a "realistic deployment" benchmark at all.

CVE-Bench is a different animal: forty real web-application CVEs, sandboxed, no advisory spoon-feeding, the agent has to find and chain the exploit itself [\[2\]](https://arxiv.org/html/2503.17332v4). Best framework: 12.5% one-day, 10% zero-day. The paper's own authors split the blame. They write that agents "frequently fail to locate the vulnerable endpoint even when given a high-level vulnerability description," and that LLM reasoning "may not be sufficient to fully understand complex vulnerabilities" [\[7\]](https://arxiv.org/html/2503.17332v4). That's the primary source itself admitting some of the failure sits with the model. I'll come back to that.

PACEbench adds a third, uglier data point: seven different LLMs, zero successful WAF bypasses, across the board [\[5\]](https://arxiv.org/html/2510.11688v1). A clean 0.000 across every model you throw at a task isn't a capability-variance signal. It's a structural wall.

The 2026 survey of 81 LLM pentest papers doesn't run new experiments of its own. It synthesizes the field and names the pattern: benchmarks lean on binary success metrics that "obscure whether reported gains reflect genuine reasoning or adaptation to specific evaluation environments" [\[8\]](https://arxiv.org/html/2607.02605v1). It cites PentestEval's 346 stage-level tasks, where mean completion sits at 0.41 and the attack-decision and exploit-generation substages specifically score around 0.25 [\[8\]](https://arxiv.org/html/2607.02605v1) — agents get partial credit for recon and triage, then stall exactly at the step requiring sustained multi-step reasoning under ambiguity. When three independently run studies — Fang et al.'s own internal control, CVE-Bench, and PACEbench — all point the same direction, the gap itself stops being debatable even though its single-number headline is sloppy.

## The failure modes underneath the numbers

Context exhaustion. Every tool call — an nmap scan, a directory listing, a verbose stack trace — eats into a fixed token budget, and raw long-context capacity doesn't save you. One memory survey found that long-context models consistently underperform purpose-built memory systems on tasks requiring selective retrieval — near-perfect scores on long-context benchmarks don't carry over once a task demands targeted lookup rather than brute-force recall [\[9\]](https://arxiv.org/html/2603.07670v1). Strobes describes a production harness that treats this as an emergency ladder with four rungs — cap tool output before it enters context, mask older outputs with summaries once a threshold hits, summarize whole conversation segments, and finally strip tool output entirely, keeping only the model's conclusions [\[10\]](https://strobes.co/blog/ai-harness-offensive-security-llm-pentest-architecture/). Flag the source honestly: that's a vendor blog post with no published benchmark numbers, a plausible design pattern rather than a validated result, unlike the academic figures elsewhere in this piece. That's not optimization. That's firefighting a window already on fire.

Long-horizon sequencing drift. Hand one LLM the entire multi-step attack chain — recon, privilege escalation, lateral movement, cleanup — and native autoregressive generation loses track of what it already tried thirty turns back. CHECKMATE and HPTSA were built to close exactly this gap; the numbers come next, but the failure itself is real and documented across multiple independent architectures [\[11\]](https://arxiv.org/abs/2512.11143)[\[12\]](https://arxiv.org/abs/2406.01637).

Hallucinated procedural steps. Free-form chain-of-thought agents, left to narrate their own next move, generate cyclical responses that repeat prior tactics or chase unused libraries that were never relevant [\[13\]](https://arxiv.org/html/2509.07939v1). The structured-attack-tree paper names this as its primary motivating failure — not a one-off glitch, a recurring pathology of open-ended generation applied to procedural, branching tasks.

False-positive noise. OpenAnt's raw candidate pool before any adversarial check sat at 376 findings across eight projects [\[14\]](https://arxiv.org/html/2606.19149). That's 376 things an analyst triages by hand if the pipeline stops there — unverified LLM output at discovery scale eats the exact analyst-hours automation was supposed to save.

Narration-forgeable approval dialogs. This is the sharpest one. The entire human-in-the-loop safety model assumes the approval dialog accurately describes the action about to run. The LITL attack breaks that assumption at the root: the dialog is narrated by the agent itself, so a compromised or simply confused agent can show a benign summary while a completely different, dangerous command actually executes [\[15\]](https://arxiv.org/abs/2606.02668). Padding, scroll-out, encoding, TOCTOU approve-one-execute-another swaps — OWASP's own community attack-pattern catalog independently documents the same technique family [\[16\]](https://arxiv.org/html/2606.02668)[\[17\]](https://community.owasp.org/attacks/Lies_in_the_Loop). Checkmarx didn't just theorize this. They showed real developers approving a padded dialog without ever noticing the dangerous action hidden underneath it [\[18\]](https://checkmarx.com/zero-post/bypassing-ai-agent-defenses-with-lies-in-the-loop/).

## Five patterns that close specific failure modes

<!-- IMAGE_PROMPT id="IMG-002" generator="ideogram4|dalle3"
Technical attack flow diagram for a security/AI practitioner blog post about LLM security agent harness engineering and consent-integrity approval gating against LITL narration attacks.

Three-zone vertical attack flow diagram depicting: Agent (malicious), Padded Dialog (malicious), Consent-Integrity Gate (trusted, critical amber highlight), Human Reviewer (trusted), Command Executor (trusted). The diagram shows two parallel flows side by side — the left sub-column shows the LITL attack path (no mediator), the right sub-column shows the hardened path (mediator in place).
Three horizontal zones stacked top to bottom: "Untrusted Agent Zone" (top zone, malicious pale red fill, red border) containing the "Agent" node and the "Padded Dialog" node side by side; "Consent-Integrity Gate" (middle zone, trusted pale teal fill, teal border, amber #FFB300 outline on the gate node itself as critical highlight) containing the "CIM Gate" node; "Trusted Execution Zone" (bottom zone, trusted pale teal fill, teal border) containing the "Human Reviewer" node on the left and the "Command Executor" node on the right.
Red dashed arrow from "Agent" down to "Padded Dialog" labeled "pads & encodes". Red dashed arrow from "Padded Dialog" bypassing the middle zone directly down to "Human Reviewer" labeled "LITL bypass" — this is the attack path shown on the left side. Solid navy arrow from "Agent" down through "CIM Gate" labeled "intercepts dialog". Solid navy arrow from "CIM Gate" down to "Human Reviewer" labeled "shows true cmd". Solid navy arrow from "Human Reviewer" right to "Command Executor" labeled "approves & runs".
Label each node with short 1–4 word text: "Agent", "Padded Dialog", "CIM Gate", "Human Reviewer", "Command Executor".

Flat design vector illustration, clean educational technical diagram, crisp geometric shapes,
editorial minimalist design, uniform line weight, no decorative elements, no texture, no noise.

Uniform flat lighting. No shadows, no highlights, no gradients, no glows, no bevels.

4:3 composition (1200×900px). Generous white space on all margins.
3 clearly labeled zones arranged top to bottom: "Untrusted Agent Zone", "Consent-Integrity Gate", "Trusted Execution Zone".

Diagram background #F4F7FA off-white.
Trusted/safe zone fills #E3F8F5 pale teal with border #00ACC1 teal.
Attacker/malicious zone fills #FFF0F0 pale red with border #F44336 red.
Neutral node fills #EEF2F6 light gray with border #263547 dark navy.
Normal flow arrows #1A2B3C dark navy solid lines.
Attack/malicious flow arrows #F44336 red dashed lines.
Critical highlight #FFB300 amber.
All text labels #1A2B3C dark navy, clean sans-serif, 1–4 words per label.

Negative prompt: photorealistic, realistic photography, camera photo, film grain,
3D CGI render, volumetric lighting, ray tracing, lens flare,
bokeh, depth of field blur, cluttered background,
decorative pattern, watermark, gradient fills, bevel, emboss,
drop shadow, glossy texture, noise, distorted shapes,
blurry, pixelated, low quality,
human faces, human hands, anime, cartoon, comic style,
rough sketch, hand-drawn lines
-->
![A three-zone vertical flow diagram showing the LITL narration attack path bypassing a human reviewer alongside the Consent-Integrity Mediator fix that intercepts and decodes padded dialogs before the human sees the true command.](/assets/media/consent-integrity-gate-flow.jpeg)
*The Consent-Integrity Gate reads ground truth from the executor, not the agent — closing the LITL narration bypass*


<!-- IMAGE_PROMPT id="IMG-001" generator="ideogram4|dalle3"
Technical comparison diagram for a security/AI practitioner blog post about LLM security agent harness engineering and five patterns that close specific failure modes.

Two-column mapping diagram depicting five failure modes (left column, neutral-fill nodes) each connected by an arrow to the corresponding pattern that closes it (right column, trusted-fill nodes). Left column nodes: Context Exhaustion (neutral), Long-Horizon Drift (neutral), Hallucinated Steps (neutral), False-Positive Noise (neutral), Forgeable Approval Dialog (malicious). Right column nodes: Ext. Memory Scoping (trusted), Planning-Layer Sep. (trusted), Attack-Tree Output (trusted), Adversarial Verify (trusted), Consent-Integrity Gate (trusted, amber critical highlight).
Two vertical columns arranged left to right: left column titled "Failure Modes" contains five stacked rounded-rectangle nodes spaced evenly top to bottom with neutral fills; right column titled "Harness Patterns" contains five corresponding rounded-rectangle nodes aligned row-for-row with trusted fills. Both columns share a common #F4F7FA background. Row labels are vertically centered inside each node.
Five solid dark navy arrows pointing left to right, one per row, connecting each failure-mode node to its corresponding pattern node: from "Context Exhaustion" to "Ext. Memory Scoping" labeled "scopes context"; from "Long-Horizon Drift" to "Planning-Layer Sep." labeled "externalizes plan"; from "Hallucinated Steps" to "Attack-Tree Output" labeled "constrains moves"; from "False-Positive Noise" to "Adversarial Verify" labeled "filters findings"; from "Forgeable Approval" to "Consent-Integrity Gate" labeled "binds approval" using a #FFB300 amber solid arrow to mark the critical path.
Label each node with short 1–4 word text: "Context Exhaustion", "Long-Horizon Drift", "Hallucinated Steps", "False-Positive Noise", "Forgeable Approval", "Ext. Memory Scoping", "Planning-Layer Sep.", "Attack-Tree Output", "Adversarial Verify", "Consent-Integrity Gate".

Flat design vector illustration, clean educational technical diagram, crisp geometric shapes,
editorial minimalist design, uniform line weight, no decorative elements, no texture, no noise.

Uniform flat lighting. No shadows, no highlights, no gradients, no glows, no bevels.

4:3 composition (1200×900px). Generous white space on all margins.
2 clearly labeled zones arranged left to right: "Failure Modes", "Harness Patterns".

Diagram background #F4F7FA off-white.
Trusted/safe zone fills #E3F8F5 pale teal with border #00ACC1 teal.
Attacker/malicious zone fills #FFF0F0 pale red with border #F44336 red.
Neutral node fills #EEF2F6 light gray with border #263547 dark navy.
Normal flow arrows #1A2B3C dark navy solid lines.
Attack/malicious flow arrows #F44336 red dashed lines.
Critical highlight #FFB300 amber.
All text labels #1A2B3C dark navy, clean sans-serif, 1–4 words per label.

Negative prompt: photorealistic, realistic photography, camera photo, film grain,
3D CGI render, volumetric lighting, ray tracing, lens flare,
bokeh, depth of field blur, cluttered background,
decorative pattern, watermark, gradient fills, bevel, emboss,
drop shadow, glossy texture, noise, distorted shapes,
blurry, pixelated, low quality,
human faces, human hands, anime, cartoon, comic style,
rough sketch, hand-drawn lines
-->
![A two-column diagram mapping five LLM agent failure modes on the left to the five harness engineering patterns that close each one on the right, connected by directional arrows.](/assets/media/harness-failure-pattern-mapping.jpeg)
*Five failure modes and the patterns engineered to close each one — the core structure of a hardened LLM pentest harness*


Call these five independently documented patterns, not a unified playbook — they come from five teams with five different threat models, and no single paper validates all five together. What's striking is how well they compound anyway.

External memory and context scoping answers context exhaustion directly. OpenAnt doesn't dump a codebase into an LLM's window and hope; it runs static reachability filtering first, cutting the analysis surface from 64,132 functions down to 2,281 reachable units — a 96.4% reduction before a single token of reasoning gets spent [\[19\]](https://arxiv.org/html/2606.19149v2). Strobes' four-layer system — capping, masking, summarizing, editing — applies the same idea to live tool-call streams, again vendor-reported rather than independently benchmarked [\[10\]](https://strobes.co/blog/ai-harness-offensive-security-llm-pentest-architecture/). Both reject the premise that a bigger window fixes this; they scope what the model sees. Not everyone agrees external memory is mandatory — HackSynth gets comparable long-horizon stability from continuous history summarization instead of persistent retrieval, a lighter option for simpler engagements [\[20\]](https://arxiv.org/html/2609.16694v1). The direction holds regardless: scope and structure working memory, don't just widen the window.

Scoped context buys a model that isn't drowning, but it doesn't stop an agent from forgetting its own plan twenty steps later. That's planning-layer separation, the most mature pattern here, because three independent architectures converge on the same move: take long-horizon sequencing off the LLM's native generation and hand it to something structurally external. PentestGPT splits reasoning, generation, and parsing into three cooperating modules around a persistent Pentesting Task Tree [\[21\]](https://github.com/GreyDGL/PentestGPT)[\[22\]](https://www.usenix.org/system/files/usenixsecurity24-deng.pdf), reporting a 228.6% task-completion increase over a GPT-3.5 baseline [\[23\]](https://arxiv.org/abs/2308.06782). That number is against a 2023 GPT-3.5 baseline — expect the margin to compress against current frontier models, the same pattern STT shows against GPT-4 later in this piece. CHECKMATE goes fully classical: the LLM perceives scan output, translates it into PDDL predicates, and also executes the simple tactical actions the planner selects, but a symbolic planner, not the LLM, owns sequencing — beating Claude Code by over 20% on benchmark success while cutting time and cost by more than half [\[11\]](https://arxiv.org/abs/2512.11143)[\[24\]](https://arxiv.org/pdf/2512.11143). Execution-layer validation — confirming the LLM ran exactly what the planner specified, not some injected or malformed variant — remains an open concern even in this architecture; the risk a red team should worry about lives in execution, not in perception or translation. HPTSA skips PDDL for a supervisor agent orchestrating specialized subagents, reporting up to a 4.3x improvement over prior single-agent frameworks [\[12\]](https://arxiv.org/abs/2406.01637)[\[25\]](https://github.com/uiuc-kang-lab/HPTSA). hackingBuddyGPT argues for the opposite extreme — a harness in fifty lines of code — though even its own Active Directory use case ends up running a persistent planner over stateless executors, undercutting a flat anti-planning reading [\[26\]](https://github.com/ipa-lab/hackingBuddyGPT). Three different mechanisms, one insight: don't let one autoregressive pass own both the plan and the next token.

Even a well-planned agent still has to pick the next legitimate action in the moment, and that's where hallucinated procedural steps creep back in — the exact failure structured attack-tree output targets. STT constrains the LLM's next move to a predefined set of actions drawn from real pentesting flows mapped onto the MITRE ATT&CK matrix, instead of letting the model narrate its own step from scratch [\[13\]](https://arxiv.org/html/2509.07939v1). The gains aren't uniform, and you should know that going in: against Llama-3-8B, STT lifts subtask success from 13.5% to 71.8%; against Gemini-1.5, from 16.5% to 72.8% — 58.3 and 56.3 points [\[27\]](https://arxiv.org/abs/2509.07939). Against GPT-4, the baseline was already 75.7%, and STT only reaches 78.6% — a 2.9-point gain [\[27\]](https://arxiv.org/abs/2509.07939). Don't read that shrinking margin as failure. Read it as structure substituting for capability: the weaker the model's native planning instinct, the more a hard-coded constraint is worth. STT is also cheaper — 55.9% fewer prompts for successful subtasks across all three tested models [\[28\]](https://arxiv.org/pdf/2509.07939).

None of the first three patterns stop a model from confidently reporting something unexploitable — and that's where false-positive noise does real damage. Adversarial pre-verification fixes it: make the model simulate an adversary under realistic constraints — no server access, no admin credentials, browser-only — and require demonstrated harm to someone other than itself before a finding survives [\[29\]](https://arxiv.org/html/2606.19149). OpenAnt runs this against 376 initial candidates and eliminates 49.5% of them, leaving 190 confirmed vulnerabilities before dynamic testing [\[14\]](https://arxiv.org/html/2606.19149). Be precise about what that proves: analysts review roughly half as many findings — a real workload reduction, not a demonstrated improvement in engagement outcomes; nobody in this literature traces fewer false positives through to better pentest results. Structured output reinforces the same instinct on the reporting side: ZeroFalse constrains adjudication to a strict SARIF schema instead of free narrative and reaches 0.912 F1 on an OWASP Java benchmark, 0.955 F1 with perfect precision on a real-world dataset [\[30\]](https://arxiv.org/html/2510.02534). A schema-constrained finding is also just faster for a human to audit than re-derived prose [\[31\]](https://www.sonarsource.com/resources/library/sarif/).

The fifth pattern should make you uncomfortable — it's not about the exploitation pipeline, it's about the safety control bolted on top of it. Consent-integrity approval gating exists because LITL proved a human-in-the-loop checkpoint is only as trustworthy as the agent narrating it; a compromised or simply confused agent can pad the dialog, push the dangerous part off-screen, or swap what executes for what got approved [\[15\]](https://arxiv.org/abs/2606.02668)[\[16\]](https://arxiv.org/html/2606.02668). The Consent-Integrity Mediator fixes this the way digital signatures fixed "what you see is what you sign": move rendering of the thing-to-be-approved out of the untrusted component into a trusted one that reads ground truth, unwinding base64-pipe-to-shell and hex-via-printf tricks before showing the human anything [\[32\]](https://arxiv.org/html/2606.02668)[\[33\]](https://arxiv.org/html/2606.02668). What's validated here is shell-command abuse specifically — GTFOBins. If your harness also issues HTTP requests, file writes, or browser actions, each needs its own ground-truth parser; the trusted-renderer concept generalizes, the deobfuscation logic does not. Against the GTFOBins corpus of trusted-tool-abuse commands, it catches 90% [\[34\]](https://arxiv.org/html/2606.02668). That 90% is measured against a known, static corpus of abuse techniques — it says nothing about resilience to an attacker who knows the mediator's deobfuscation logic and crafts novel bypasses for the remaining 10%. Red-team your own consent mediator before you trust it; GTFOBins is a floor, not a ceiling, on adversarial robustness. A sibling assurance-properties framework names the same idea "enforced authorization" and "tamper-evident accountability," finding a consistent assurance gap across production harnesses including PentestGPT [\[35\]](https://arxiv.org/abs/2609.22664), and an independent Loopjacking paper converges on the same binding principle from a different angle [\[36\]](https://arxiv.org/html/2609.21081v1).

Here's the honest cost: run the same analyzer against normal usage — 28,798 commands from tldr — and it marks 87% uninspectable too [\[37\]](https://arxiv.org/html/2606.02668). A separate formal analysis of approval-gate design shows why that matters: escalating too much to human review is formally worse than an intermediate rate, because reviewers fatigue and start rubber-stamping [\[38\]](https://arxiv.org/html/2606.08919v1). A gate that prompts on 87% of everything doesn't train humans to scrutinize — it trains them to click through [\[39\]](https://www.developersdigest.tech/blog/approval-fatigue-agent-security-bug). The pattern is sound. This prototype's calibration isn't production-ready.

## The counter-argument you should sit with

CVE-Bench's own authors don't frame their results as a scaffolding problem. They say plainly that agents "frequently fail to locate the vulnerable endpoint even when given a high-level vulnerability description," and that LLM reasoning "may not be sufficient to fully understand complex vulnerabilities" [\[7\]](https://arxiv.org/html/2503.17332v4). That's the benchmark's authors, not an outside critic, assigning real weight to the model itself. The structured-attack-tree numbers back them up in one specific place: GPT-4's gain from STT guidance is only 2.9 percentage points, against 58.3 and 56.3 points for Llama-3 and Gemini [\[27\]](https://arxiv.org/abs/2509.07939). If harness engineering explained everything, a stronger harness should lift every model by a comparable margin. It doesn't. The harness-versus-capability debate is genuinely unsettled in this literature, and I won't pretend CVE-Bench's authors are simply wrong.

Here's my rebuttal, and it isn't a dodge. "Frequently fail to locate the vulnerable endpoint" is a textbook exploration and context-scoping failure. OpenAnt's reachability filtering illustrates the underlying principle — cutting the search space by 96.4% before the model ever has to find anything in the dark [\[19\]](https://arxiv.org/html/2606.19149v2) — but be precise about what it actually is: a white-box technique, an AST-based call graph run over source code. CVE-Bench agents work black-box, over HTTP, against a live target with no source to analyze, so the technique doesn't transfer directly. What does transfer is PentestGPT's Task Tree, which tracks which endpoints the agent has already probed and which remain unexplored, bounding a black-box search the same way reachability filtering bounds a white-box one [\[21\]](https://github.com/GreyDGL/PentestGPT)[\[22\]](https://www.usenix.org/system/files/usenixsecurity24-deng.pdf). Different mechanism, same principle: bound the search space deliberately, don't rely on the model to explore unaided. The shrinking STT margin on GPT-4 isn't evidence against the thesis, either. It's the strongest evidence for it. The pattern works hardest exactly where the model's own planning instinct is weakest — exactly what you'd expect if structure were compensating for a capability gap rather than papering over one. Both CHECKMATE and HPTSA report their gains without claiming to have swapped in a stronger model [\[11\]](https://arxiv.org/abs/2512.11143)[\[12\]](https://arxiv.org/abs/2406.01637) — though neither paper explicitly confirms exact model-version parity between harness and baseline, so treat "same model, restructured harness" as the papers' own framing, not an independently verified control. Even on that more cautious reading, the harness lever is doing real work.

## What this means for the team building now

Don't start by chasing a bigger or newer model. Start by auditing where your agent's context actually goes once a tool call returns. Most teams stuff raw output straight into the window and call it done — precisely the failure OpenAnt and Strobes independently engineered around [\[19\]](https://arxiv.org/html/2606.19149v2)[\[10\]](https://strobes.co/blog/ai-harness-offensive-security-llm-pentest-architecture/). Second, separate your planner from your executor before anything else. PentestGPT, CHECKMATE, and HPTSA arrived at that same structural decision from three different directions, and the convergence itself is the signal [\[21\]](https://github.com/GreyDGL/PentestGPT)[\[11\]](https://arxiv.org/abs/2512.11143)[\[12\]](https://arxiv.org/abs/2406.01637). Third, treat structured output as a floor, not a nice-to-have — if you're running frontier models, budget for a smaller STT accuracy gain and look for the win in token efficiency instead, because 55.9% fewer prompts is real money at scale [\[28\]](https://arxiv.org/pdf/2509.07939). Fourth, false-positive filtering buys you analyst time, not proof of better engagement outcomes — say that plainly to whoever is funding the program, because the literature hasn't closed that loop yet. Fifth, if your harness has a human-approval gate, stop assuming the vendor hardened it by default. Checkmarx proved real developers approve padded dialogs without noticing [\[18\]](https://checkmarx.com/zero-post/bypassing-ai-agent-defenses-with-lies-in-the-loop/), and a correctly calibrated consent mediator is harder to build than it looks — a 90% catch rate is good, but 87% over-prompting on normal traffic is a second problem you now have to solve on top of the first [\[34\]](https://arxiv.org/html/2606.02668)[\[37\]](https://arxiv.org/html/2606.02668). And sixth — don't skip this one: none of these five patterns have been evaluated against actively-defended targets. PACEbench's zero-across-the-board WAF result stands untouched by any harness pattern in this piece — if your engagements involve live detection/response, treat that as a separate, unsolved problem, not something planner separation or context scoping will fix [\[5\]](https://arxiv.org/html/2510.11688v1). The teams that win this aren't the ones with the newest model behind the harness. They're the ones treating their own harness as adversarial terrain.

---

## References

- \[1\] [ChatGPT 4 can exploit 87% of one-day vulnerabilities](https://www.ibm.com/think/insights/chatgpt-4-exploits-87-percent-one-day-vulnerabilities)
- \[2\] [CVE-Bench: A Benchmark for AI Agents' Ability to Exploit Real-World Web Application Vulnerabilities](https://arxiv.org/html/2503.17332v4)
- \[3\] [CVE-Bench: A Benchmark for AI Agents' Ability to Exploit Real-World Web Application Vulnerabilities](https://arxiv.org/abs/2503.17332)
- \[4\] [AI Pentesting Agents 2026: The Rise of 39+ Tools Tested](https://appsecsanta.com/research/ai-pentesting-agents-2026)
- \[5\] [PACEbench: A Framework for Evaluating Practical AI Cyber-Exploitation Capabilities](https://arxiv.org/html/2510.11688v1)
- \[6\] [ChatGPT 4 can exploit 87% of one-day vulnerabilities](https://www.ibm.com/think/insights/chatgpt-4-exploits-87-percent-one-day-vulnerabilities)
- \[7\] [CVE-Bench: A Benchmark for AI Agents' Ability to Exploit Real-World Web Application Vulnerabilities](https://arxiv.org/html/2503.17332v4)
- \[8\] [A Survey of LLM-Driven Penetration Testing: Taxonomy, Co-Evolution, and Open Challenges](https://arxiv.org/html/2607.02605v1)
- \[9\] [Memory for Autonomous LLM Agents: Mechanisms, Evaluation, and Emerging Frontiers](https://arxiv.org/html/2603.07670v1)
- \[10\] [Pentest Harness Architecture for AI Security](https://strobes.co/blog/ai-harness-offensive-security-llm-pentest-architecture/)
- \[11\] [Automated Penetration Testing with LLM Agents and Classical Planning (CHECKMATE)](https://arxiv.org/abs/2512.11143)
- \[12\] [Teams of LLM Agents can Exploit Zero-Day Vulnerabilities (HPTSA)](https://arxiv.org/abs/2406.01637)
- \[13\] [Guided Reasoning in LLM-Driven Penetration Testing Using Structured Attack Trees](https://arxiv.org/html/2509.07939v1)
- \[14\] [OpenAnt: LLM-Powered Vulnerability Discovery Through Code Decomposition, Adversarial Verification, and Dynamic Testing](https://arxiv.org/html/2606.19149)
- \[15\] [What You Approve Is What Executes: Consent Integrity for Black-Box LLM Agents](https://arxiv.org/abs/2606.02668)
- \[16\] [What You Approve Is What Executes: Consent Integrity for Black-Box LLM Agents](https://arxiv.org/html/2606.02668)
- \[17\] [HITL Dialog Forging (aka Lies-in-the-Loop)](https://community.owasp.org/attacks/Lies_in_the_Loop)
- \[18\] [Bypassing AI Agent Defenses With Lies-In-The-Loop](https://checkmarx.com/zero-post/bypassing-ai-agent-defenses-with-lies-in-the-loop/)
- \[19\] [OpenAnt: LLM-Powered Vulnerability Discovery Through Code Decomposition, Adversarial Verification, and Dynamic Testing](https://arxiv.org/html/2606.19149v2)
- \[20\] [Toward Secure AI-Powered Penetration Testing Agents: Security Threats, Guardrails, and Architectural Perspectives](https://arxiv.org/html/2609.16694v1)
- \[21\] [GreyDGL/PentestGPT: Automated Penetration Testing Agentic Framework Powered by Large Language Models](https://github.com/GreyDGL/PentestGPT)
- \[22\] [PentestGPT: Evaluating and Harnessing Large Language Models for Automated Penetration Testing (USENIX Security 2024)](https://www.usenix.org/system/files/usenixsecurity24-deng.pdf)
- \[23\] [PentestGPT: Evaluating and Harnessing Large Language Models for Automated Penetration Testing](https://arxiv.org/abs/2308.06782)
- \[24\] [Automated Penetration Testing with LLM Agents and Classical Planning](https://arxiv.org/pdf/2512.11143)
- \[25\] [uiuc-kang-lab/HPTSA GitHub repository](https://github.com/uiuc-kang-lab/HPTSA)
- \[26\] [ipa-lab/hackingBuddyGPT: Helping Ethical Hackers use LLMs in 50 Lines of Code or less](https://github.com/ipa-lab/hackingBuddyGPT)
- \[27\] [Guided Reasoning in LLM-Driven Penetration Testing Using Structured Attack Trees](https://arxiv.org/abs/2509.07939)
- \[28\] [Guided Reasoning in LLM-Driven Penetration Testing Using Structured Attack Trees](https://arxiv.org/pdf/2509.07939)
- \[29\] [OpenAnt: LLM-Powered Vulnerability Discovery Through Code Decomposition, Adversarial Verification, and Dynamic Testing](https://arxiv.org/html/2606.19149)
- \[30\] [ZeroFalse: Improving Precision in Static Analysis with LLMs](https://arxiv.org/html/2510.02534)
- \[31\] [The complete guide to SARIF: Standardizing static analysis results](https://www.sonarsource.com/resources/library/sarif/)
- \[32\] [What You Approve Is What Executes: Consent Integrity for Black-Box LLM Agents](https://arxiv.org/html/2606.02668)
- \[33\] [What You Approve Is What Executes: Consent Integrity for Black-Box LLM Agents](https://arxiv.org/html/2606.02668)
- \[34\] [What You Approve Is What Executes: Consent Integrity for Black-Box LLM Agents](https://arxiv.org/html/2606.02668)
- \[35\] [Autonomous Penetration-Testing Assurance Framework (five assurance properties paper)](https://arxiv.org/abs/2609.22664)
- \[36\] [Loopjacking: Hijacking Human-in-the-Loop Approval](https://arxiv.org/html/2609.21081v1)
- \[37\] [What You Approve Is What Executes: Consent Integrity for Black-Box LLM Agents](https://arxiv.org/html/2606.02668)
- \[38\] [Oversight Has a Capacity: Calibrating Agent Guards to a Subjective, Fatiguing Human](https://arxiv.org/html/2606.08919v1)
- \[39\] [Approval Fatigue Is an Agent Security Bug](https://www.developersdigest.tech/blog/approval-fatigue-agent-security-bug)
