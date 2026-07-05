# Behavioural interview prep

You said you'll likely replace the stories with new ones from Propylon. So this doc is built to help you write those, with the old Irish Life / Infobip stories kept below as ready-made backup (Irish Life is still relevant — Citco is financial services).

## 1. Positioning — the narrative to lead with

Get ahead of the seam between your CV (leans on the AI stack) and your history (mostly C#/.NET + financial services). Honest, confident frame:

> "Full-stack engineer, ~5 years — strongest in C#/.NET and in financial-services software at Irish Life, plus document-systems work at Propylon across a Python/Django + Postgres backend. FastAPI, RAG and the agentic side are newer to me, so rather than just claim the keywords I built DocIntel end-to-end to actually understand it. I'd rather tell you what I built and where it's rough than oversell it."

**Why it works:** referral-backed hire → they value a real signal over a polished claim; their spec explicitly wants responsible-AI awareness / knowing the limitations, so humility reads as a strength.

**Rule:** only say "I built X" for things you can defend under one follow-up.

## 2. STAR framework + the competencies to cover

Keep each story ~60–90 seconds. Your own note from being an interviewer: they have zero context, keep it relevant, don't get bogged in detail, hit the key phrases.

**Structure:** Situation (1 line of context) → Task (what was on you) → Action (what you specifically did) → Result (outcome, ideally measurable).

For each topic, here's what interviewers are typically scoring for — mid-level fullstack, so they're weighing technical depth against ownership/communication more than they would for junior:

### Difficult troubleshooting / debugging

**Looking for:** systematic method, not luck. Did you form a hypothesis, isolate variables, use logs/tools rather than guess-and-check? For third-level support specifically, they want to see you distinguishing symptom from root cause, and communicating status while still investigating (not going dark for hours). Bonus: did you leave the system better (added logging, alert, runbook) so the next person doesn't repeat the hunt.

### Ownership end-to-end

**Looking for:** did you drive it from ambiguous ask to production, including the unglamorous parts (deployment, monitoring, telling stakeholders it's done)? They're checking you don't need a project manager assigning you sub-tasks. Mention a point where you made a judgment call without waiting for permission.

**Story: condensed amendment PDF (Propylon).** From the bill draft request portal, depending on the state of the amendment request in its process (is in final review), allow the user to create a PDF version of the amendment document with only the pages they choose.

**S — Situation**

The legislative drafting team was manually producing "condensed" amendment PDFs: open the Word document, File → Print → Microsoft Print to PDF, manually select page 1 plus whatever pages the committee needed, save it next to the original file. Their downstream automation then picked it up. The ask came in as "can you automate this?" — no spec, no wireframe, just that sentence and a manual process to observe.

**T — Task**

I owned the feature end-to-end across a Django web app (BDR) and a VSTO Word add-in (BDE). My job was to go from that ambiguous ask to a working, integrated feature — including figuring out the architecture, writing the spec, and building both sides.

**A — Action**

The first judgment call was PDF generation strategy. The obvious route was server-side conversion (LibreOffice headless), but I identified early that Word's pagination engine and the manual output had to match exactly — these are legal documents, and a line wrapping differently on page 4 would matter to a committee. I decided to route generation through the VSTO add-in instead, using Word's own print engine. That decision wasn't in any requirements doc; I made it and moved on.

The second architectural problem was communication. The `ms-word:` URI scheme that launches Word can't carry custom parameters, so I couldn't just pass page numbers in the URL. I traced how the existing combo-amendment feature solved the same problem: it writes structured data into the docx as custom XML properties, and the add-in reads them on document open. I extended that pattern — added two new ContentFields to the model (`CONDENSEDPAGES`, `CONDENSEDCREATED`), wrote a Django view that validates the user's page spec, canonicalises it (always include page 1, dedup, sort), writes it into the docx metadata, and returns the Word launch URI. The add-in reads the fields on open, generates the PDF, and sets `CONDENSEDCREATED=true` so it doesn't re-run on incidental re-opens.

On the front end I added a gated toggle to the existing Create AIC dialog — enabled only when the amendment is at Final Drafter Review or To Committee — with client-side validation (Bootstrap validator pattern attribute) and a tooltip showing the expected format. The form's action URL swaps dynamically depending on which toggle is active.

One unglamorous catch I found during testing: `MetadataManager.getMetadataValue` throws an exception when a property is absent rather than returning an empty string. Existing AICs in the datastore wouldn't have my new fields. I added explicit initialisation of both fields to `""` when any AIC is created, so the add-in always finds them present. That's the kind of thing that would have surfaced as a silent crash in production on a doc nobody had regenerated.

For the automation log — another thing that wasn't in the ask — I noticed that writing the PDF into the datastore creates an LrmsRevision, which is exactly what the team's existing copy automation monitors. No extra instrumentation needed; the trigger was already there by construction.

**R — Result**

The feature shipped as a self-contained end-to-end flow: user selects pages in the browser, Word opens, PDF appears next to the docx, automation picks it up — no manual steps remaining. I wrote the full cross-system spec for the C# side so the add-in work could proceed in a separate session without re-deriving the architecture decisions.

### Non-technical / cross-functional stakeholders

**Looking for:** translation skill — could you explain a technical constraint (e.g. why a data migration takes downtime) in terms a client or business user cares about? For Citco this maps directly to your world: fund accountants/clients who use the LRMS system but aren't engineers. Show you adapted the message, not just repeated jargon slower.

### Initiative / improving a process or introducing a technology

**Looking for:** did you spot the problem before someone told you to fix it, and did you justify the investment (why this tool/process, what was the cost of the status quo)? They want evidence you don't just execute tickets. Your DocIntel/Python upskilling angle fits here if you built something with it, not just learned it.

**Story A — process improvement: client bug ticket quality (Montana).**

- **S** — Clients raised bug tickets with just a screenshot and a short description of what's wrong and what they expected. That was rarely enough to act on — we'd need logs, the files where the issue occurred, exact replication steps, full-screen screenshots.
- **T** — Every under-specified ticket meant a clarification round-trip with the client before investigation could even start, delaying fixes. Nobody owned fixing the intake process; I took it on.
- **A** — Defined what an actionable ticket needs (logs, affected files, exact replication steps, full-screen screenshots) and got that baked into how clients raise tickets, so the information arrives up front instead of being chased afterwards.
- **R** — Far less back-and-forth with the client; tickets arrive actionable and investigation starts immediately instead of after a clarification cycle.

**Story B — introducing a technology: AI-assisted coding practices (Montana).**

- **S** — The team was adopting AI coding assistants (Copilot / Claude) ad hoc — no shared conventions, so the tools gave verbose, unfocused output and occasionally did damage, like editing VS autogenerated designer code in the VSTO project, which leads to unexpected results.
- **T** — Make the tooling reliable and cheap enough to be worth using, rather than each person burning credits re-explaining context every session.
- **A** — Introduced instruction files (copilot-instructions / claude.md) encoding how the assistant should work: concise by default, verbose only when necessary (caveman rule); never touch VS autogenerated code like designer files; follow our coding styles. Used graphify to generate an index map of the codebase architecture so the assistant navigates instead of re-reading everything, with a rule to prompt me to regenerate it after structural changes.
- **R** — The team burned through their credits much slower, and the assistant's output became safer and more consistent — no more designer-file edits.

### Prioritising under competing deadlines

**Looking for:** your reasoning method (impact vs effort, risk, who's blocked), and that you communicated tradeoffs upward rather than silently dropping something. A red flag answer is "I just worked more hours" — they want a prioritization framework.

### A difficult decision or tradeoff

**Looking for:** that you can articulate the alternatives you didn't pick and why, own the outcome (including if it was imperfect), and show judgment under uncertainty/incomplete information. This is the one where "and it worked out great" answers feel weakest — a tradeoff with a real cost you accepted is more convincing.

### Team collaboration

**Looking for:** a concrete instance of making someone else more effective (unblocking, reviewing, pairing, filling a gap) — not just "we worked well together." They're listening for humility (crediting others) balanced with your specific contribution.

### Learning something new quickly

**Looking for:** your learning method under time pressure (how you scoped what to learn vs skip, who/what you used as resources) and proof of application, not just comprehension. A demo/output is stronger than "I read the docs."

**General STAR mechanics they'll be grading regardless of topic:** concrete S/T (specific system, not "a project"), your individual actions in first person, and a result with a number or verifiable outcome where possible.

## 3. Backup stories (ready to use — from the earlier prep)

- **Difficult troubleshooting — mortgage-protection quote (Irish Life).** On production support, a broker couldn't generate a quote for a product I didn't know. I got a dummy account from the broker-app team, replicated it in QA, debugged, and found the customer wasn't eligible for the product. Told the broker so they could regenerate correctly, and raised a Jira ticket to add an eligibility check to prevent recurrence. → Cross-team, methodical, fixed the cause not the symptom. (Financial services + third-level support — very Citco.)

- **Performance investigation (Irish Life / OLS).** Integrating OLS into the OIL portal, loads were slow. I measured method execution times, found a service returning far more fund-price data than the page needed, and switched to an endpoint returning only the customer's invested-fund prices using params already in session. → Measurement-driven; doubles as your "slow API" technical answer.

- **Initiative + data-driven decision (Infobip / Ansible).** Dev-environment setup was slow and inconsistent. I measured setup time via Jira, learned Ansible, estimated the build effort, and took the cost/benefit to a sprint retro to get it planned in. → Matches Citco's "investigate new tech and present for architectural review" almost word-for-word.

- **Ownership + stakeholders + domain (One Irish Life / OLS).** I owned the OLS integration into the unified portal: gathered requirements, built the API endpoints and deep links, wired Azure AD SSO into every interaction, and supported QA's test plans. → Financial services, API/SOA, auth/SSO, ownership in one story.

- **Team player under pressure.** A colleague going on holiday wasn't going to finish his task before a strict deadline; my own was on track, so I took a handover and finished his ticket. → "Effective independently and collaboratively."

- **Competing priorities + communication.** Asked to add a webchat feature mid-project, both due the same deadline — I raised it early in stand-up rather than quietly slipping, and the team split the sprint work. Both shipped on time.

> ⚠️ **Difficult-decision story to finish:** the customer-similarity bug near a deadline (flag it and miss vs. let QA catch it). Good story, but your notes don't record the outcome — write the honest ending (almost certainly: you flagged it early with a justification, matching your own stated principle of never hiding delays) before you use it.

## 4. Questions to ask them

- What does the MOS platform stack look like today, and where are you introducing agentic workflows — production or exploratory?
- Split between greenfield features vs. enhancing/supporting the existing treasury/collateral/trade platforms?
- Are you using Bedrock / AgentCore in production yet, or building toward it?
- What does a typical release cycle and the third-level-support rota look like?
- Team size/structure — who would I work most closely with, and who's the Technical Lead I'd report to?
- What does success in the first six months look like?
- What's team culture and retention like?

## 5. Strengths / weaknesses / career goals

**Strengths:** learns new tech fast and is eager to (DocIntel is the proof); organised, juggles concurrent tasks to hit deadlines; communicates well across technical and non-technical people. Back each with a one-line example.

**Weakness (framed with a fix):** "I like to fully understand why something works before I'm confident with it, so I front-load learning — which is exactly why I built DocIntel rather than just reading about RAG." (Avoid the old "afraid of being wrong" framing.)

**Career goals:** deepen technical range across new stacks (Python/FastAPI/AI is exactly that), more building services from scratch and extending existing systems, and moving toward designing/evaluating architectures. Maps well to a role that's both new features and platform support.

## Quick cheat sheet

- Lead with financial services + strong debugging/ownership; frame the AI stack as the gap I built DocIntel to close.
- Never claim past what you've built and understand. "I'm building X" is a strong answer.
- Best backup story: the mortgage-protection troubleshoot.
- Finish the customer-similarity story's ending before the interview.
- Keep answers tight — give the signal, not the saga.
