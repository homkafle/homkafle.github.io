---
layout: post
title: "A2A Reproduced OAuth's Original Sin"
categories: [Artificial Intelligence, Security]
tags: [ai-security, llm-security, agentic-ai, reproduced, oauth, original]
fullview: false
description: ""
comments: false
---

Open the A2A protocol's enterprise-readiness docs and you'll find a sentence that should stop any identity engineer cold: "Authorization logic is specific to the agent's implementation, the data it handles, and applicable enterprise policies." [E-457d-002] That's not a gap in the documentation. That's the documentation.

One scope note before anything else: this piece is about agent-to-agent trust — orchestrator-to-subagent handoffs, A2A-style delegation chains, the protocol layer where one autonomous agent decides to trust another. It is not about single-agent tool poisoning, the MCP server-trust problem where a compromised tool definition hijacks one agent's own context. Different attack surface, different failure mode, different fix. Keep them separate or you'll reach for the wrong mitigation.

## The OAuth parallel, specifically

I've spent a chunk of my career building and auditing OAuth and OIDC deployments, so I'm allergic to loose historical analogies. Here's the precise one. RFC 6749 states plainly that "access token attributes and the methods used to access protected resources are beyond the scope of this specification" [E-457d-005] — a deliberate deferral of token format and transport to companion specs. OAuth Core 1.0a's gap was different: "By itself, OAuth does not provide any method for scoping the access rights granted to a Consumer" [E-457d-006], a hole 2.0 actually patched by adding the scope parameter 1.0a lacked. Two specs, two different defects. What they share is the shape, not the hole: 2.0 added scope, then left scope *enforcement* — what a value actually permits, who checks it — entirely to each Authorization Server and Resource Server to define for itself. That's the real parallel to A2A: not "no scoping," but "scoping with no mandated enforcement." Eran Hammer, who led the OAuth 2.0 working group and then resigned from it, wrote the quiet part out loud: OAuth 2.0 "at the hands of most developers" was "likely to produce insecure implementations" [E-457d-007]. That wasn't a prediction about bad actors. It was a prediction about a spec that handed enforcement to whoever shipped last.

Now read A2A's own documentation again. "A2A does not define how the authorization must be performed. Without a clear definition, the system becomes vulnerable to potential security problems, including authorization creep" [E-457d-003]. A separate independent write-up states it even more bluntly: authorization "is left pretty much up to implementers," with remote agents responsible for access control after authentication completes [E-457d-301]. And Semgrep's security engineers, reading the spec straight, land on the same line: "Authorization is implementation-defined: To be clear — this is not the protocol's job!" [E-457d-204]. That's three independent parties converging on one sentence. Sound familiar?

Now here's a sentence that cuts the other way, and I'm not going to pretend it doesn't exist. TrueFoundry's comparison guide describes A2A's "Enterprise-Grade Security" as something that "enforces strong authentication and authorization, aligning with OpenAPI security schemes to ensure safe agent collaboration across platforms" [E-457d-306]. Read uncritically, that flatly contradicts everything above. Read carefully, it's consistent with it — because it's collapsing two different words into one. A2A's authentication layer really is strong: the spec lets an Agent Card declare OAuth2, OpenID Connect, mTLS, or any enterprise identity scheme an implementer wants to require [E-457d-008]. One caveat worth keeping in your pocket: OAuth2 alone isn't authentication — it's a delegated-authorization protocol whose access tokens carry no standardized, verifiable identity claims, so declaring bare "OAuth2" without an OIDC-style ID token reproduces the exact "OAuth-as-authentication" mistake that OIDC was built in 2014 to fix. Authentication answers one question — who are you? Authorization answers a different one — what are you allowed to do, and who's checking? A2A has a real, extensible answer to the first question and no protocol-mandated answer to the second; that's exactly what its own documentation says [E-457d-002][E-457d-003]. TrueFoundry isn't lying about authentication. It's using the word "authorization" as decoration on a sentence about authentication, and that imprecision is a trap — the same trap that let people call OAuth 2.0 "secure" for a decade while Eran Hammer was resigning over exactly this gap.

The Agent Card compounds it. A2A's spec requires a client to authenticate using whatever scheme the Agent Card itself declares [E-457d-008] — but the Agent Card is typically fetched anonymously, unsigned, by default, before any of that authentication happens [E-457d-010]. Red Hat's own security guidance concedes the built-in protections are "often insufficient on its own" [E-457d-009]. You're authenticating against a document nobody verified.

## A2A is the documented case, not the only case

Here's where I'll push back on my own framing before someone else does it for me. A2A isn't uniquely broken. It's the best-documented instance of a pattern that shows up across the agent ecosystem. An arXiv comparative threat model spanning MCP, A2A, Agora, and ANP found that A2A's scoping "is not strictly enforced; indeed, it is left to the identity and authorization infrastructure underneath" [E-457d-304] — and it's not alone. A security comparison of LangGraph, AutoGen, and CrewAI found that none of the three frameworks "enforce identity-bound policy on the outbound model call by default" [E-457d-305]. The deferral is architectural habit, not a Google-specific oversight.

That said, A2A sits at the worse end of its own peer set. MCP, for comparison, mandates OAuth 2.1 with PKCE for HTTP-based remote server deployments, including Authorization Server Metadata and Dynamic Client Registration — local, stdio-transport servers sit outside that mandate [E-457d-302]. A2A requires none of that, for any deployment mode. If you're triaging which agent protocol deserves the closest audit first, start here.

## Eleven ways to break a spec-compliant agent

<!-- IMAGE_PROMPT id="IMG-001" generator="ideogram4|dalle3"
Technical attack flow for a security/AI practitioner blog post about A2A protocol trust gaps and spec-compliant agent vulnerabilities.

A2A protocol vulnerability attack flow diagram: depicting Orchestrator (trusted A2A caller initiating task delegation), Malicious Client (malicious A2A peer that knows a valid contextId and injects tasks into shared context), Agent Context (neutral shared task context targeted by injection, critical element), Downstream Agent (neutral subagent receiving delegated tasks where the original principal identity is lost), Rogue Agent (malicious subagent advertising unattested capability claims to redirect orchestration), Credential Provider (neutral identity verifier that cannot confirm original principal across hops).
Three horizontal zones stacked top to bottom: "Caller Layer" (top) containing Orchestrator on the left and Malicious Client on the right; "A2A Protocol Layer" (middle) containing Agent Context in the center and Credential Provider on the right; "Subagent Layer" (bottom) containing Downstream Agent on the left and Rogue Agent on the right.
Solid dark navy arrow from Orchestrator down to Agent Context labeled "Delegates Task". Red dashed arrow from Malicious Client down to Agent Context labeled "Context Injection". Solid dark navy arrow from Agent Context down to Downstream Agent labeled "Task Handoff". Red dashed arrow from Downstream Agent right to Credential Provider labeled "Identity Lost". Red dashed arrow from Rogue Agent upward to Orchestrator labeled "False Skill Claim". Amber (#FFB300) highlight border on Agent Context as the single most critical injection point.
Label each node with short 1-4 word text: "Orchestrator", "Malicious Client", "Agent Context", "Downstream Agent", "Rogue Agent", "Credential Provider".

Flat design vector illustration, clean educational technical diagram, crisp geometric shapes,
editorial minimalist design, uniform line weight, no decorative elements, no texture, no noise.

Uniform flat lighting. No shadows, no highlights, no gradients, no glows, no bevels.

4:3 composition (1200x900px). Generous white space on all margins.
3 clearly labeled zones arranged top to bottom: "Caller Layer", "A2A Protocol Layer", "Subagent Layer".

Diagram background #F4F7FA off-white.
Trusted/safe zone fills #E3F8F5 pale teal with border #00ACC1 teal.
Attacker/malicious zone fills #FFF0F0 pale red with border #F44336 red.
Neutral node fills #EEF2F6 light gray with border #263547 dark navy.
Normal flow arrows #1A2B3C dark navy solid lines.
Attack/malicious flow arrows #F44336 red dashed lines.
Critical highlight #FFB300 amber.
All text labels #1A2B3C dark navy, clean sans-serif, 1-4 words per label.

Negative prompt: photorealistic, realistic photography, camera photo, film grain,
3D CGI render, volumetric lighting, ray tracing, lens flare,
bokeh, depth of field blur, cluttered background,
decorative pattern, watermark, gradient fills, bevel, emboss,
drop shadow, glossy texture, noise, distorted shapes,
blurry, pixelated, low quality,
human faces, human hands, anime, cartoon, comic style,
rough sketch, hand-drawn lines
-->
![Three-zone attack flow diagram showing how a Malicious Client exploits the A2A protocol layer via Cross-Client Context Injection, how Multi-Hop Identity Loss prevents a Credential Provider from verifying the original caller, and how a Rogue Agent's false skill claims redirect an Orchestrator toward unintended exfiltration paths.](/assets/media/a2a-protocol-attack-vectors.png)
*Three spec-compliant A2A attack vectors: context injection, identity loss at hop boundaries, and unattested skill claims*


A2ABreak, the ACSAC 2026 artifact analyzing A2A, identified eleven distinct protocol-level vulnerabilities — each "exploitable by a specification-compliant adversary without requiring any implementation flaw" [E-457d-011]. Read that phrase twice. These aren't bugs in somebody's Python. They're gaps in the state machine itself.

Three are worth naming. Cross-Client Context Injection: "any authenticated client that knows a valid contextId can inject a new task into that context" [E-457d-014]. Multi-Hop Identity Loss: "downstream agents and credential providers have no means to verify the original principal" once a request crosses more than one hop [E-457d-015]. Unattested Skill Claims: nothing stops "rogue agents advertising unattested capability claims," opening a path to exfiltration through agents that simply lied about what they could do [E-457d-016].

I want to be precise about how these were validated, because the paper is precise about it too. Automated adversarial analysis against a formal finite-state-machine model of the spec systematically identified all eleven findings, cross-validated against independent expert manual review at 73.3% precision and an 84.6% F1 score [E-457d-020]. The researchers did not demonstrate them against a live, deployed A2A implementation [E-457d-018][E-457d-019]. That distinction matters and I'm not going to blur it: these are flaws systematically identified via automated analysis over a formal model of the spec's state machine, cross-checked by human reviewers — not field exploits. The honest claim is that a compliant implementation of the spec, built exactly to the letter, inherits all eleven gaps by construction. That's arguably worse than a code bug — you can't patch your way out of it without changing the protocol.

## Why propagation changes the blast-radius math

<!-- IMAGE_PROMPT id="IMG-002" generator="ideogram4|dalle3"
Technical attack flow for a security/AI practitioner blog post about A2A protocol trust gaps and multi-agent prompt propagation blast radius.

Five-agent prompt infection pipeline diagram: depicting Injected Document (malicious source document carrying an embedded self-replicating attack instruction, entry point), Tool Agent (neutral document-reading first-hop agent that processes the injected document and becomes the initial infection vector, critical element), Strategist (neutral second-hop planning agent that receives the propagated malicious instruction from Tool Agent), Summarizer (neutral third-hop condensing agent that carries the infection forward), Editor (neutral fourth-hop refinement agent that continues propagating the infected context), Writer (neutral fifth-hop output agent that produces final output under malicious instruction influence).
Six nodes arranged in a single horizontal pipeline row left to right with no stacked zones: Injected Document at the far left, then Tool Agent, Strategist, Summarizer, Editor, and Writer at the far right.
Red dashed arrow from Injected Document to Tool Agent labeled "Inject". Red dashed arrow from Tool Agent to Strategist labeled "Propagates". Red dashed arrow from Strategist to Summarizer labeled "Propagates". Red dashed arrow from Summarizer to Editor labeled "Propagates". Red dashed arrow from Editor to Writer labeled "Propagates". Amber (#FFB300) highlight border on Tool Agent as the critical first infection hop.
Label each node with short 1-4 word text: "Injected Doc", "Tool Agent", "Strategist", "Summarizer", "Editor", "Writer".

Flat design vector illustration, clean educational technical diagram, crisp geometric shapes,
editorial minimalist design, uniform line weight, no decorative elements, no texture, no noise.

Uniform flat lighting. No shadows, no highlights, no gradients, no glows, no bevels.

16:9 composition (1200x675px). Generous white space on all margins.
No zones, single horizontal pipeline row arranged left to right.

Diagram background #F4F7FA off-white.
Trusted/safe zone fills #E3F8F5 pale teal with border #00ACC1 teal.
Attacker/malicious zone fills #FFF0F0 pale red with border #F44336 red.
Neutral node fills #EEF2F6 light gray with border #263547 dark navy.
Normal flow arrows #1A2B3C dark navy solid lines.
Attack/malicious flow arrows #F44336 red dashed lines.
Critical highlight #FFB300 amber.
All text labels #1A2B3C dark navy, clean sans-serif, 1-4 words per label.

Negative prompt: photorealistic, realistic photography, camera photo, film grain,
3D CGI render, volumetric lighting, ray tracing, lens flare,
bokeh, depth of field blur, cluttered background,
decorative pattern, watermark, gradient fills, bevel, emboss,
drop shadow, glossy texture, noise, distorted shapes,
blurry, pixelated, low quality,
human faces, human hands, anime, cartoon, comic style,
rough sketch, hand-drawn lines
-->
![Left-to-right pipeline diagram showing a malicious injected document infecting five sequential agents — Tool Agent, Strategist, Summarizer, Editor, and Writer — with red dashed arrows depicting how the malicious instruction self-replicates across every hop in the chain.](/assets/media/prompt-infection-pipeline.png)
*One injected document, five infected agents: how prompt infection compounds blast radius across every hop*


Here's the part of the argument I won't assert without showing my work, because the evidence bundle doesn't contain a clean quantitative OAuth-vs-A2A severity comparison, and claiming one exists would be dishonest.

What the evidence does contain is this: the "Prompt Infection" study, published at ESORICS 2025 [E-457d-106], built a five-agent pipeline — a document-reading tool agent feeding a strategist, summarizer, editor, and writer — and tested 360 unique attack/instruction pairs across it [E-457d-102]. The researchers describe the result as "a novel attack where malicious prompts self-replicate across interconnected agents, behaving much like a computer virus" [E-457d-101]. Self-replicating injection beat non-replicating injection by 13.92% on GPT-4o and by 209% on GPT-3.5 [E-457d-103]. Messaging architecture matters too: local messaging, where agents see only partial histories, cut the self-replicating attack's success rate by roughly 20% compared to global, shared-history messaging [E-457d-103]. Even there, a single uncompromised agent in the chain was enough to kill the infection outright — and the reason it spreads as far as it does otherwise is that the infection prompt manipulates both the model and its own importance-scoring mechanism, creating a feedback loop that keeps the attack alive [E-457d-104].

Sit with that last detail. In an OAuth system, a mis-scoped token is a single-hop failure — one resource server honors a token it shouldn't, and the blast radius is bounded by what that token can reach (true as long as the token is properly audience-restricted; before RFC 8707 and sender-constrained tokens like DPoP standardized that binding, bearer tokens had their own history of being replayed across resource servers that had no business accepting them). Even granting that messy history, A2A's propagation mechanism is structurally worse: it needs no second mis-scoped credential at all. In a multi-agent chain built on an unenforced trust boundary, an injected instruction rides the conversation itself, hop to hop, compounding whatever capability each subsequent agent happens to hold. OAuth's failure mode is a stuck door. A2A's failure mode, absent per-hop re-validation, is a stuck door that unlocks the next three doors on its way through. That's the structural argument for a worse blast radius — not a severity number, a mechanism.

The Cloud Security Alliance's MAESTRO framework [E-457d-115], applied directly to A2A, rates unauthorized agent impersonation via spoofed Agent Cards as "High (Medium likelihood, high impact)" [E-457d-109], citing "weak authentication mechanisms, reliance on easily spoofed Agent Cards" [E-457d-110] and recommending cryptographic verification against a server's public key or DID as an add-on mitigation — not a built-in requirement [E-457d-111]. A named industry framework reaches the same conclusion as the formal FSM analysis, independently.

## Sandboxing is real. It's also not enough.

I don't want to be the guy who dismisses agent-level defenses because they don't fit a tidy protocol-reform thesis. They're real, they're deployed, and they help. The Prompt Infection authors' own proposed defense, LLM Tagging, prepends a marker identifying a message as agent-originated rather than user-originated, and the authors report it "significantly mitigates infection spread" — when combined with other safeguards [E-457d-307]. Execution sandboxing contains compromised agent behavior even though it "cannot prevent prompt injection" outright [E-457d-310]. Design patterns like the action-selector, which constrains an agent to an LLM-modulated "switch statement" over a fixed set of permitted actions, constrain what an injected instruction can actually do [E-457d-309].

None of that is in dispute. What's in dispute is sufficiency. Palo Alto Networks' own security researchers put the underlying limit plainly: "a system prompt can describe a boundary for an AI agent but cannot enforce one" [E-457d-215]. The arXiv paper on design patterns for securing LLM agents makes the same point about architecture, not just prompts: "Use a combination of design patterns to achieve robust security; no single pattern is likely to suffice across all threat models or use cases" [E-457d-308]. That's not a hedge — it's a direct statement that agent-level isolation, however well-built, caps out. And the reason it caps out is structural: sandboxing and capability restriction operate inside a single agent's execution boundary. Cross-Client Context Injection, Multi-Hop Identity Loss, and spoofed Agent Cards all operate at the inter-agent protocol layer, the wire between agents — a layer that sandboxing was never designed to inspect. You can sandbox an agent into a cage and it will still accept a forged Agent Card at the front door, because the cage doesn't check IDs.

## What actually closes the gap

A2ABreak's own framing of the problem is the clearest statement of the fix: the spec treats "context ownership, delegation provenance, capability attestation, and credential scoping as implementation concerns rather than protocol-level invariants," and that choice "propagate[s] directly as exploitable vulnerabilities in any compliant deployment" [E-457d-201]. Move those four things from implementer-optional to protocol-mandatory and you close most of the eleven findings at once.

Some of this movement is already happening, and credit where it's due. A2A v1.0, released in 2026, formalized cryptographically signed Agent Cards [E-457d-211] — a real step up from the v0.3-era state where signing was supported but not enforced [E-457d-203]. Production deployments inside Azure AI Foundry, Amazon Bedrock AgentCore, and Salesforce Agentforce reportedly combine signed Agent Cards with OAuth-scoped skill authorization and transport-layer encryption [E-457d-212]. That describes marketed capability, not an independently audited posture — none of it confirms per-hop re-authorization or principal propagation across delegation chains is solved on any of these platforms. Separately, an academic proposal for hardening A2A introduces ephemeral, scoped delegation tokens and explicit consent orchestration specifically to limit exposure across hops [E-457d-209].

Red Hat recommends mutual TLS between agents to cut replay and impersonation risk [E-457d-205]. Scope that precisely, because this is exactly where false confidence creeps in: mTLS is a hop-layer control, not a delegation-chain control. It authenticates the channel between agent A and agent B for that one connection — it tells agent C nothing about who B is actually acting on behalf of when B calls C next. That's Multi-Hop Identity Loss, the A2ABreak finding named above, and mTLS doesn't touch it. Closing that gap needs a mechanism that carries the original principal forward across every hop — OAuth 2.0 Token Exchange, specified in RFC 8693, with actor and may_act claims that preserve who originally made the request, is the closest existing standard built for exactly this problem. Nothing in the current A2A spec requires anything like it.

Here's my one caveat, and it's load-bearing: "formalized" is not the same word as "mandatory." The Linux Foundation's own announcement says v1.0 formalized signed Agent Cards [E-457d-211] — it does not say every compliant implementation must sign and verify them. Until a primary source confirms that distinction has closed, treat Agent Card signing the way you'd treat an optional security header: present where someone bothered to turn it on, absent everywhere else. The same implementer-discretion pattern that produced OAuth's "road to hell" is still sitting inside the current spec version, just wearing a nicer label.

If you're building or auditing a multi-agent deployment on A2A or anything architecturally similar, don't wait for the next spec revision to save you. Enforce per-hop re-authorization using a delegation mechanism that carries the original principal forward — OAuth 2.0 Token Exchange (RFC 8693) with actor and may_act claims is the closest existing standard; a workload-identity framework like SPIFFE/SPIRE, paired with a policy decision point at every hop, is a reasonable alternative. Either beats a fresh credential check that re-authenticates the immediate caller while losing track of who originated the request. Verify Agent Card signatures even when the protocol merely permits it — and only trust that verification if the key comes from an independent anchor: a CA-issued certificate, a DNS-bound key, or an independently resolvable DID, not the same unauthenticated channel that served the card. Scope delegation tokens to the single task at hand, not the session. Pair that with the agent-level controls — sandboxing, capability restriction, tagging — that already work. None of this is a blanket mandate: if your agent only touches read-only APIs inside a trusted internal network, with no adversarial data anywhere in its path, lighter-weight guardrails are proportionate and full protocol hardening is overkill [E-457d-214]. But that's a narrow carve-out, not the shape of most production multi-agent systems being shipped right now. OAuth took roughly a decade of real-world breach data to force that kind of discipline onto implementers who'd rather have shipped without it. Multi-agent systems, with instructions that propagate on their own across every hop you forgot to lock down, don't have a decade to spare.