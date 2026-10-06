---
layout: post
title: "A2A Reproduced OAuth's Original Sin — Then Gave It a Multi-Hop Blast Radius"
categories: [Artificial Intelligence, Security]
tags: [ai-security, llm-security, agentic-ai, reproduced, oauth, original, then, gave, multi, blast]
fullview: false
description: "Open the A2A protocol's enterprise-readiness docs and you'll find a sentence that should stop any identity engineer cold: \"Authorization logic is specific to the agent's implementation, the data it…"
comments: false
---

Open the A2A protocol's enterprise-readiness docs and you'll find a sentence that should stop any identity engineer cold: "Authorization logic is specific to the agent's implementation, the data it handles, and applicable enterprise policies." [\[1\]](https://a2a-protocol.org/latest/topics/enterprise-ready/) That's not a gap in the documentation. That's the documentation.

One scope note before anything else: this piece is about agent-to-agent trust — orchestrator-to-subagent handoffs, A2A-style delegation chains, the protocol layer where one autonomous agent decides to trust another. It is not about single-agent tool poisoning, the MCP server-trust problem where a compromised tool definition hijacks one agent's own context. Different attack surface, different failure mode, different fix. Keep them separate or you'll reach for the wrong mitigation.

## The OAuth parallel, specifically

I've spent a chunk of my career building and auditing OAuth and OIDC deployments, so I'm allergic to loose historical analogies. Here's the precise one. RFC 6749 states plainly that "access token attributes and the methods used to access protected resources are beyond the scope of this specification" [\[2\]](https://www.rfc-editor.org/info/rfc6749/) — a deliberate deferral of token format and transport to companion specs. OAuth Core 1.0a's gap was different: "By itself, OAuth does not provide any method for scoping the access rights granted to a Consumer" [\[3\]](https://oauth.net/core/1.0a/), a hole 2.0 actually patched by adding the scope parameter 1.0a lacked. Two specs, two different defects. What they share is the shape, not the hole: 2.0 added scope, then left scope *enforcement* — what a value actually permits, who checks it — entirely to each Authorization Server and Resource Server to define for itself. That's the real parallel to A2A: not "no scoping," but "scoping with no mandated enforcement." Eran Hammer, who led the OAuth 2.0 working group and then resigned from it, wrote the quiet part out loud: OAuth 2.0 "at the hands of most developers" was "likely to produce insecure implementations" [\[4\]](https://gist.github.com/nckroy/dd2d4dfc86f7d13045ad715377b6a48f). That wasn't a prediction about bad actors. It was a prediction about a spec that handed enforcement to whoever shipped last.

Now read A2A's own documentation again. "A2A does not define how the authorization must be performed. Without a clear definition, the system becomes vulnerable to potential security problems, including authorization creep" [\[5\]](https://developers.redhat.com/articles/2025/08/19/how-enhance-agent2agent-security). A separate independent write-up states it even more bluntly: authorization "is left pretty much up to implementers," with remote agents responsible for access control after authentication completes [\[6\]](https://www.descope.com/blog/post/mcp-vs-a2a-auth). And Semgrep's security engineers, reading the spec straight, land on the same line: "Authorization is implementation-defined: To be clear — this is not the protocol's job!" [\[7\]](https://semgrep.dev/blog/2025/a-security-engineers-guide-to-the-a2a-protocol/). That's three independent parties converging on one sentence. Sound familiar?

Now here's a sentence that cuts the other way, and I'm not going to pretend it doesn't exist. TrueFoundry's comparison guide describes A2A's "Enterprise-Grade Security" as something that "enforces strong authentication and authorization, aligning with OpenAPI security schemes to ensure safe agent collaboration across platforms" [\[8\]](https://www.truefoundry.com/blog/mcp-vs-a2a). Read uncritically, that flatly contradicts everything above. Read carefully, it's consistent with it — because it's collapsing two different words into one. A2A's authentication layer really is strong: the spec lets an Agent Card declare OAuth2, OpenID Connect, mTLS, or any enterprise identity scheme an implementer wants to require [\[9\]](https://a2a-protocol.org/latest/specification/). One caveat worth keeping in your pocket: OAuth2 alone isn't authentication — it's a delegated-authorization protocol whose access tokens carry no standardized, verifiable identity claims, so declaring bare "OAuth2" without an OIDC-style ID token reproduces the exact "OAuth-as-authentication" mistake that OIDC was built in 2014 to fix. Authentication answers one question — who are you? Authorization answers a different one — what are you allowed to do, and who's checking? A2A has a real, extensible answer to the first question and no protocol-mandated answer to the second; that's exactly what its own documentation says [\[1\]](https://a2a-protocol.org/latest/topics/enterprise-ready/)[\[5\]](https://developers.redhat.com/articles/2025/08/19/how-enhance-agent2agent-security). TrueFoundry isn't lying about authentication. It's using the word "authorization" as decoration on a sentence about authentication, and that imprecision is a trap — the same trap that let people call OAuth 2.0 "secure" for a decade while Eran Hammer was resigning over exactly this gap.

The Agent Card compounds it. A2A's spec requires a client to authenticate using whatever scheme the Agent Card itself declares [\[9\]](https://a2a-protocol.org/latest/specification/) — but the Agent Card is typically fetched anonymously, unsigned, by default, before any of that authentication happens [\[10\]](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-to-agent-authentication). Red Hat's own security guidance concedes the built-in protections are "often insufficient on its own" [\[11\]](https://developers.redhat.com/articles/2025/08/19/how-enhance-agent2agent-security). You're authenticating against a document nobody verified.

## A2A is the documented case, not the only case

Here's where I'll push back on my own framing before someone else does it for me. A2A isn't uniquely broken. It's the best-documented instance of a pattern that shows up across the agent ecosystem. An arXiv comparative threat model spanning MCP, A2A, Agora, and ANP found that A2A's scoping "is not strictly enforced; indeed, it is left to the identity and authorization infrastructure underneath" [\[12\]](https://arxiv.org/html/2602.11327v1) — and it's not alone. A security comparison of LangGraph, AutoGen, and CrewAI found that none of the three frameworks "enforce identity-bound policy on the outbound model call by default" [\[13\]](https://www.deepinspect.ai/blog/agentic-ai-frameworks-security-comparison). The deferral is architectural habit, not a Google-specific oversight.

That said, A2A sits at the worse end of its own peer set. MCP, for comparison, mandates OAuth 2.1 with PKCE for HTTP-based remote server deployments, including Authorization Server Metadata and Dynamic Client Registration — local, stdio-transport servers sit outside that mandate [\[14\]](https://www.descope.com/blog/post/mcp-vs-a2a-auth). A2A requires none of that, for any deployment mode. If you're triaging which agent protocol deserves the closest audit first, start here.

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


A2ABreak, the ACSAC 2026 artifact analyzing A2A, identified eleven distinct protocol-level vulnerabilities — each "exploitable by a specification-compliant adversary without requiring any implementation flaw" [\[15\]](https://arxiv.org/abs/2609.10871). Read that phrase twice. These aren't bugs in somebody's Python. They're gaps in the state machine itself.

Three are worth naming. Cross-Client Context Injection: "any authenticated client that knows a valid contextId can inject a new task into that context" [\[16\]](https://arxiv.org/html/2609.10871). Multi-Hop Identity Loss: "downstream agents and credential providers have no means to verify the original principal" once a request crosses more than one hop [\[17\]](https://arxiv.org/html/2609.10871). Unattested Skill Claims: nothing stops "rogue agents advertising unattested capability claims," opening a path to exfiltration through agents that simply lied about what they could do [\[18\]](https://arxiv.org/abs/2609.10871).

I want to be precise about how these were validated, because the paper is precise about it too. Automated adversarial analysis against a formal finite-state-machine model of the spec systematically identified all eleven findings, cross-validated against independent expert manual review at 73.3% precision and an 84.6% F1 score [\[19\]](https://arxiv.org/html/2609.10871). The researchers did not demonstrate them against a live, deployed A2A implementation [\[20\]](https://pith.science/paper/2609.10871)[\[21\]](https://zenodo.org/records/22868506). That distinction matters and I'm not going to blur it: these are flaws systematically identified via automated analysis over a formal model of the spec's state machine, cross-checked by human reviewers — not field exploits. The honest claim is that a compliant implementation of the spec, built exactly to the letter, inherits all eleven gaps by construction. That's arguably worse than a code bug — you can't patch your way out of it without changing the protocol.

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

What the evidence does contain is this: the "Prompt Infection" study, published at ESORICS 2025 [\[22\]](https://dl.acm.org/doi/10.1007/978-3-032-16092-8_28), built a five-agent pipeline — a document-reading tool agent feeding a strategist, summarizer, editor, and writer — and tested 360 unique attack/instruction pairs across it [\[23\]](https://arxiv.org/html/2410.07283). The researchers describe the result as "a novel attack where malicious prompts self-replicate across interconnected agents, behaving much like a computer virus" [\[24\]](https://arxiv.org/abs/2410.07283). Self-replicating injection beat non-replicating injection by 13.92% on GPT-4o and by 209% on GPT-3.5 [\[25\]](https://www.alphaxiv.org/abs/2410.07283). Messaging architecture matters too: local messaging, where agents see only partial histories, cut the self-replicating attack's success rate by roughly 20% compared to global, shared-history messaging [\[25\]](https://www.alphaxiv.org/abs/2410.07283). Even there, a single uncompromised agent in the chain was enough to kill the infection outright — and the reason it spreads as far as it does otherwise is that the infection prompt manipulates both the model and its own importance-scoring mechanism, creating a feedback loop that keeps the attack alive [\[26\]](https://www.alphaxiv.org/abs/2410.07283).

Sit with that last detail. In an OAuth system, a mis-scoped token is a single-hop failure — one resource server honors a token it shouldn't, and the blast radius is bounded by what that token can reach (true as long as the token is properly audience-restricted; before RFC 8707 and sender-constrained tokens like DPoP standardized that binding, bearer tokens had their own history of being replayed across resource servers that had no business accepting them). Even granting that messy history, A2A's propagation mechanism is structurally worse: it needs no second mis-scoped credential at all. In a multi-agent chain built on an unenforced trust boundary, an injected instruction rides the conversation itself, hop to hop, compounding whatever capability each subsequent agent happens to hold. OAuth's failure mode is a stuck door. A2A's failure mode, absent per-hop re-validation, is a stuck door that unlocks the next three doors on its way through. That's the structural argument for a worse blast radius — not a severity number, a mechanism.

The Cloud Security Alliance's MAESTRO framework [\[27\]](https://cloudsecurityalliance.org/blog/2025/02/06/agentic-ai-threat-modeling-framework-maestro), applied directly to A2A, rates unauthorized agent impersonation via spoofed Agent Cards as "High (Medium likelihood, high impact)" [\[28\]](https://cloudsecurityalliance.org/blog/2025/04/30/threat-modeling-google-s-a2a-protocol-with-the-maestro-framework), citing "weak authentication mechanisms, reliance on easily spoofed Agent Cards" [\[29\]](https://cloudsecurityalliance.org/blog/2025/04/30/threat-modeling-google-s-a2a-protocol-with-the-maestro-framework) and recommending cryptographic verification against a server's public key or DID as an add-on mitigation — not a built-in requirement [\[30\]](https://cloudsecurityalliance.org/blog/2025/04/30/threat-modeling-google-s-a2a-protocol-with-the-maestro-framework). A named industry framework reaches the same conclusion as the formal FSM analysis, independently.

## Sandboxing is real. It's also not enough.

I don't want to be the guy who dismisses agent-level defenses because they don't fit a tidy protocol-reform thesis. They're real, they're deployed, and they help. The Prompt Infection authors' own proposed defense, LLM Tagging, prepends a marker identifying a message as agent-originated rather than user-originated, and the authors report it "significantly mitigates infection spread" — when combined with other safeguards [\[31\]](https://www.emergentmind.com/papers/2410.07283). Execution sandboxing contains compromised agent behavior even though it "cannot prevent prompt injection" outright [\[32\]](https://www.augmentcode.com/guides/agent-execution-sandbox). Design patterns like the action-selector, which constrains an agent to an LLM-modulated "switch statement" over a fixed set of permitted actions, constrain what an injected instruction can actually do [\[33\]](https://arxiv.org/html/2506.08837v2).

None of that is in dispute. What's in dispute is sufficiency. Palo Alto Networks' own security researchers put the underlying limit plainly: "a system prompt can describe a boundary for an AI agent but cannot enforce one" [\[34\]](https://unit42.paloaltonetworks.com/navigating-security-tradeoffs-ai-agents/). The arXiv paper on design patterns for securing LLM agents makes the same point about architecture, not just prompts: "Use a combination of design patterns to achieve robust security; no single pattern is likely to suffice across all threat models or use cases" [\[35\]](https://arxiv.org/html/2506.08837v2). That's not a hedge — it's a direct statement that agent-level isolation, however well-built, caps out. And the reason it caps out is structural: sandboxing and capability restriction operate inside a single agent's execution boundary. Cross-Client Context Injection, Multi-Hop Identity Loss, and spoofed Agent Cards all operate at the inter-agent protocol layer, the wire between agents — a layer that sandboxing was never designed to inspect. You can sandbox an agent into a cage and it will still accept a forged Agent Card at the front door, because the cage doesn't check IDs.

## What actually closes the gap

A2ABreak's own framing of the problem is the clearest statement of the fix: the spec treats "context ownership, delegation provenance, capability attestation, and credential scoping as implementation concerns rather than protocol-level invariants," and that choice "propagate[s] directly as exploitable vulnerabilities in any compliant deployment" [\[36\]](https://arxiv.org/abs/2609.10871). Move those four things from implementer-optional to protocol-mandatory and you close most of the eleven findings at once.

Some of this movement is already happening, and credit where it's due. A2A v1.0, released in 2026, formalized cryptographically signed Agent Cards [\[37\]](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year) — a real step up from the v0.3-era state where signing was supported but not enforced [\[38\]](https://semgrep.dev/blog/2025/a-security-engineers-guide-to-the-a2a-protocol/). Production deployments inside Azure AI Foundry, Amazon Bedrock AgentCore, and Salesforce Agentforce reportedly combine signed Agent Cards with OAuth-scoped skill authorization and transport-layer encryption [\[39\]](https://tyk.io/learning-center/a2a-protocol-architecture-and-technical-specification/). That describes marketed capability, not an independently audited posture — none of it confirms per-hop re-authorization or principal propagation across delegation chains is solved on any of these platforms. Separately, an academic proposal for hardening A2A introduces ephemeral, scoped delegation tokens and explicit consent orchestration specifically to limit exposure across hops [\[40\]](https://arxiv.org/abs/2505.12490).

Red Hat recommends mutual TLS between agents to cut replay and impersonation risk [\[41\]](https://developers.redhat.com/articles/2025/08/19/how-enhance-agent2agent-security). Scope that precisely, because this is exactly where false confidence creeps in: mTLS is a hop-layer control, not a delegation-chain control. It authenticates the channel between agent A and agent B for that one connection — it tells agent C nothing about who B is actually acting on behalf of when B calls C next. That's Multi-Hop Identity Loss, the A2ABreak finding named above, and mTLS doesn't touch it. Closing that gap needs a mechanism that carries the original principal forward across every hop — OAuth 2.0 Token Exchange, specified in RFC 8693, with actor and may_act claims that preserve who originally made the request, is the closest existing standard built for exactly this problem. Nothing in the current A2A spec requires anything like it.

Here's my one caveat, and it's load-bearing: "formalized" is not the same word as "mandatory." The Linux Foundation's own announcement says v1.0 formalized signed Agent Cards [\[37\]](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year) — it does not say every compliant implementation must sign and verify them. Until a primary source confirms that distinction has closed, treat Agent Card signing the way you'd treat an optional security header: present where someone bothered to turn it on, absent everywhere else. The same implementer-discretion pattern that produced OAuth's "road to hell" is still sitting inside the current spec version, just wearing a nicer label.

If you're building or auditing a multi-agent deployment on A2A or anything architecturally similar, don't wait for the next spec revision to save you. Enforce per-hop re-authorization using a delegation mechanism that carries the original principal forward — OAuth 2.0 Token Exchange (RFC 8693) with actor and may_act claims is the closest existing standard; a workload-identity framework like SPIFFE/SPIRE, paired with a policy decision point at every hop, is a reasonable alternative. Either beats a fresh credential check that re-authenticates the immediate caller while losing track of who originated the request. Verify Agent Card signatures even when the protocol merely permits it — and only trust that verification if the key comes from an independent anchor: a CA-issued certificate, a DNS-bound key, or an independently resolvable DID, not the same unauthenticated channel that served the card. Scope delegation tokens to the single task at hand, not the session. Pair that with the agent-level controls — sandboxing, capability restriction, tagging — that already work. None of this is a blanket mandate: if your agent only touches read-only APIs inside a trusted internal network, with no adversarial data anywhere in its path, lighter-weight guardrails are proportionate and full protocol hardening is overkill [\[42\]](https://www.datadoghq.com/blog/securing-ai-agents-guardrail-placement/). But that's a narrow carve-out, not the shape of most production multi-agent systems being shipped right now. OAuth took roughly a decade of real-world breach data to force that kind of discipline onto implementers who'd rather have shipped without it. Multi-agent systems, with instructions that propagate on their own across every hop you forgot to lock down, don't have a decade to spare.

---

## References

- \[1\] [Enterprise Features - A2A Protocol](https://a2a-protocol.org/latest/topics/enterprise-ready/)
- \[2\] [RFC 6749: The OAuth 2.0 Authorization Framework - RFC Editor](https://www.rfc-editor.org/info/rfc6749/)
- \[3\] [OAuth Core 1.0 Revision A](https://oauth.net/core/1.0a/)
- \[4\] [OAuth 2.0 and the Road to Hell (reproduction of Eran Hammer's post)](https://gist.github.com/nckroy/dd2d4dfc86f7d13045ad715377b6a48f)
- \[5\] [How to enhance Agent2Agent (A2A) security - Red Hat Developer](https://developers.redhat.com/articles/2025/08/19/how-enhance-agent2agent-security)
- \[6\] [Comparing Auth Approaches in MCP and A2A](https://www.descope.com/blog/post/mcp-vs-a2a-auth)
- \[7\] [A Security Engineer's Guide to the A2A Protocol - Semgrep](https://semgrep.dev/blog/2025/a-security-engineers-guide-to-the-a2a-protocol/)
- \[8\] [MCP vs A2A: Compare Single-Agent & Multi-Agent Protocols](https://www.truefoundry.com/blog/mcp-vs-a2a)
- \[9\] [Agent2Agent (A2A) Protocol Specification](https://a2a-protocol.org/latest/specification/)
- \[10\] [Agent2Agent (A2A) authentication - Microsoft Foundry - Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-to-agent-authentication)
- \[11\] [How to enhance Agent2Agent (A2A) security - Red Hat Developer](https://developers.redhat.com/articles/2025/08/19/how-enhance-agent2agent-security)
- \[12\] [Security Threat Modeling for Emerging AI-Agent Protocols: A Comparative Analysis of MCP, A2A, Agora, and ANP](https://arxiv.org/html/2602.11327v1)
- \[13\] [Agentic AI Framework Security: Comparing How LangGraph, AutoGen, and CrewAI Handle the Model-Call Boundary](https://www.deepinspect.ai/blog/agentic-ai-frameworks-security-comparison)
- \[14\] [Comparing Auth Approaches in MCP and A2A](https://www.descope.com/blog/post/mcp-vs-a2a-auth)
- \[15\] [A2ABreak: Systematic Security Analysis of the A2A Protocol](https://arxiv.org/abs/2609.10871)
- \[16\] [A2ABreak: Systematic Security Analysis of the A2A Protocol (HTML)](https://arxiv.org/html/2609.10871)
- \[17\] [A2ABreak: Systematic Security Analysis of the A2A Protocol (HTML)](https://arxiv.org/html/2609.10871)
- \[18\] [A2ABreak: Systematic Security Analysis of the A2A Protocol](https://arxiv.org/abs/2609.10871)
- \[19\] [A2ABreak: Systematic Security Analysis of the A2A Protocol (HTML)](https://arxiv.org/html/2609.10871)
- \[20\] [A2ABreak: Systematic Security Analysis of the A2A Protocol · Pith Review](https://pith.science/paper/2609.10871)
- \[21\] [arlotfi79/A2ABreak: A2ABreak v1.0.0 — ACSAC 2026 Artifact](https://zenodo.org/records/22868506)
- \[22\] [Prompt Infection: LLM-to-LLM Prompt Injection within Multi-agent Systems - Computer Security. ESORICS 2025 International Workshops](https://dl.acm.org/doi/10.1007/978-3-032-16092-8_28)
- \[23\] [Prompt Infection: LLM-to-LLM Prompt Injection within Multi-Agent Systems (HTML)](https://arxiv.org/html/2410.07283)
- \[24\] [Prompt Infection: LLM-to-LLM Prompt Injection within Multi-Agent Systems](https://arxiv.org/abs/2410.07283)
- \[25\] [Prompt Infection: LLM-to-LLM Prompt Injection within Multi-Agent Systems - alphaXiv](https://www.alphaxiv.org/abs/2410.07283)
- \[26\] [Prompt Infection: LLM-to-LLM Prompt Injection within Multi-Agent Systems - alphaXiv](https://www.alphaxiv.org/abs/2410.07283)
- \[27\] [Agentic AI Threat Modeling Framework: MAESTRO - CSA](https://cloudsecurityalliance.org/blog/2025/02/06/agentic-ai-threat-modeling-framework-maestro)
- \[28\] [Threat Modeling Google's A2A Protocol - CSA](https://cloudsecurityalliance.org/blog/2025/04/30/threat-modeling-google-s-a2a-protocol-with-the-maestro-framework)
- \[29\] [Threat Modeling Google's A2A Protocol - CSA](https://cloudsecurityalliance.org/blog/2025/04/30/threat-modeling-google-s-a2a-protocol-with-the-maestro-framework)
- \[30\] [Threat Modeling Google's A2A Protocol - CSA](https://cloudsecurityalliance.org/blog/2025/04/30/threat-modeling-google-s-a2a-protocol-with-the-maestro-framework)
- \[31\] [LLM Prompt Infection in Multi-Agent Systems](https://www.emergentmind.com/papers/2410.07283)
- \[32\] [What Is an Agent Execution Sandbox?](https://www.augmentcode.com/guides/agent-execution-sandbox)
- \[33\] [Design Patterns for Securing LLM Agents against Prompt Injections](https://arxiv.org/html/2506.08837v2)
- \[34\] [Navigating Security Tradeoffs of AI Agents - Unit 42, Palo Alto Networks](https://unit42.paloaltonetworks.com/navigating-security-tradeoffs-ai-agents/)
- \[35\] [Design Patterns for Securing LLM Agents against Prompt Injections](https://arxiv.org/html/2506.08837v2)
- \[36\] [A2ABreak: Systematic Security Analysis of the A2A Protocol](https://arxiv.org/abs/2609.10871)
- \[37\] [A2A Protocol Surpasses 150 Organizations, Lands in Major Cloud Platforms, and Sees Enterprise Production Use in First Year](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year)
- \[38\] [A Security Engineer's Guide to the A2A Protocol - Semgrep](https://semgrep.dev/blog/2025/a-security-engineers-guide-to-the-a2a-protocol/)
- \[39\] [A2A Protocol: The Definitive Agent-to-Agent Guide](https://tyk.io/learning-center/a2a-protocol-architecture-and-technical-specification/)
- \[40\] [Improving Google A2A Protocol: Protecting Sensitive Data and Mitigating Unintended Harms in Multi-Agent Systems](https://arxiv.org/abs/2505.12490)
- \[41\] [How to enhance Agent2Agent (A2A) security - Red Hat Developer](https://developers.redhat.com/articles/2025/08/19/how-enhance-agent2agent-security)
- \[42\] [Securing AI agents: Why guardrail placement is a key design decision - Datadog](https://www.datadoghq.com/blog/securing-ai-agents-guardrail-placement/)
