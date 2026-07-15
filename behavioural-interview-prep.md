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

**Story: public member assets deleted on committee removal (Montana, CM-308).**

**S — Situation**

A client bug ticket reported that removing a public member from an interim committee in the LRMS committee-management system was wiping that person's assets, not just their membership on that one committee.

**T — Task**

Troubleshoot the ticket: reproduce it, find the root cause, and fix it — knowing going in that "removal" touches both the Aurelia frontend and the Django backend, so the cause could be on either side.

**A — Action**

I set up a dummy public member on a dummy interim committee in a dev/test environment so I could reproduce on demand without touching real data. Reproducing confirmed it: removing the PM from the committee wiped more than that one membership. I checked logs first to see which request/endpoint fired on removal, then stepped through the code on both sides to trace the flow. On the frontend (`edit-committee.js`) I found removed public members were tracked in a set and their UUIDs sent to the backend on save as `removedPublicMembers`. On the backend (`committee.py`, `set_committee_members`) I found a special-cased block that took that list and ran `User.objects.filter(uuid__in=removed_public_members).delete()` — an unscoped hard delete of the member's global `User` record. Every other member removal in the same function only deletes the `CommitteePosition` join row for that specific committee. So the bug wasn't in the "removal" feature at all — it was a conflation of two different operations: "remove from this committee" was implemented as "delete this person," which cascaded away all their assets across the whole system, not just the one committee link.

**R — Result**

Fixed by removing the `removedPublicMembers` hard-delete path entirely — public member removal now goes through the same `CommitteePosition`-only deletion as every other member type, scoped correctly to the one committee. I used the same change to add drag-and-drop for adding public members to a committee, since I was already in that code. Ticket resolved, and the fix removed a live data-loss risk rather than just patching the symptom.

**🔥 Likely pushback**

- **Q: Why did that hard-delete special case exist in the first place — was it intentional?** A: I didn't dig into the original commit history — the priority was closing the live data-loss risk, not archaeology. My best guess is it was written for a genuinely different use case (deleting a public member's account outright) and got wired into the wrong removal flow. Worth a follow-up ticket, not worth blocking the fix on.
- **Q: How did you know your fix wouldn't break some other flow relying on that cascade?** A: Every other member-type removal in the same function was already doing the `CommitteePosition`-only deletion — mine was the outlier, not the norm. Making public members match the existing pattern was lower-risk than the special case ever was.
- **Q: This sounds like exactly the kind of thing a unit test should catch. Why wasn't there one?** A: There wasn't coverage on the `removedPublicMembers` path specifically, which is exactly how it shipped and survived. That's a real gap I'd own, not explain away.

### Ownership end-to-end

**Looking for:** did you drive it from ambiguous ask to production, including the unglamorous parts (deployment, monitoring, telling stakeholders it's done)? They're checking you don't need a project manager assigning you sub-tasks. Mention a point where you made a judgment call without waiting for permission.

**Story: tabbed-table HTML export (Montana).** The Word add-in's bill export was losing column alignment for "tabbed tables" — legislative content (e.g. appropriations tables) built from paragraphs with runs of tab characters rather than real Word tables — so exported HTML rendered them as mangled, unaligned text.

**S — Situation**

Bill exports from the Word add-in (`mt-bde-law-making`) render real `Word.Table` objects into HTML fine, but tabbed pseudo-tables — used throughout appropriations bills — collapsed into flat, misaligned text on export. No one had specified a fix; it surfaced as a gap I found while working the export pipeline.

**T — Task**

I owned it end-to-end across two systems: the C# Word add-in and the Python document API (`mt-doc-api`) it talks to — architecture, detection logic, conversion logic, and the splice back into the export, plus the tests and rollout safety net.

**A — Action**

The first judgment call was where the work should live. I split it: detection stays in C#, conversion happens in Python. The add-in already walks paragraphs section-by-section and already has `Word.Range.WordOpenXML` to hand off a self-contained XML fragment per table, so C# keeping ownership of "where are the tables" avoided duplicating that section-boundary logic in Python and kept the Python side a pure, unit-testable `fragment → html` converter with no document-model coupling.

The second problem was detection itself — there's no dedicated paragraph style marking these as tables, so it has to be structural. "Any paragraph containing a tab" false-positives on ordinary statute text like `(i)\ttext`. I pulled real bill samples and found the actual dividing line: rows have ≥2 interior tab groups, certification lines and statute paragraphs don't. I pinned that rule against both a positive and negative real-world sample before writing the parser.

The highest-stakes call was correctness, not architecture: these documents carry amendment marking — strike/underline for legal insertions and deletions — entirely through character styles, not run properties. A converter that ignored that would silently delete legally meaningful markup. I mapped the specific style names to the same span markup the existing real-table export path already emits, rather than inventing a new representation.

Throughout, I kept the change reversible: the whole feature sits behind a flag, and if the API call fails for any reason the export falls back to the old (visually broken but functioning) behavior — an export must never fail because of this feature. I also required the toggle-off path to produce byte-identical output to pre-change exports, as a standing regression guard.

**R — Result**

Shipped. Tabbed tables now render as proper HTML tables — same CSS classes and structure as real Word tables, so the two are visually indistinguishable in the export — while the existing real-table export path and non-participating exports were verified unchanged.

**🔥 Likely pushback**

- **Q: Why route through the add-in instead of a standalone headless docx→html converter?** A: The add-in already has Word's own view of the document — section bookmarks, ranges, revision state — for free. Rebuilding that from raw XML in a headless parser risked drifting from what the actual export produces. Piggybacking on the add-in's existing structure kept detection and export in sync by construction.
- **Q: What happens if `mt-doc-api` is slow or down mid-export?** A: The POST has its own short timeout, separate from the general HTTP client timeout, and any failure — timeout, 5xx, network — falls back to the pre-existing output. The feature can only fail *open* onto the old rendering, never fail the export itself.
- **Q: That ≥2-tab-groups heuristic sounds fragile — what if a real bill breaks it?** A: It's a heuristic tuned against real samples, not a proof, and I know that. That's why a misdetected or malformed table surfaces as a per-table warning and falls back to the old rendering for just that table — not a silently wrong legal document. Fail visibly and locally was the deliberate design choice.

### Non-technical / cross-functional stakeholders

**Looking for:** translation skill — could you explain a technical constraint (e.g. why a data migration takes downtime) in terms a client or business user cares about? For Citco this maps directly to your world: fund accountants/clients who use the LRMS system but aren't engineers. Show you adapted the message, not just repeated jargon slower.

**Story: budget bill versioning display (Propylon).** *Also covers: working with stakeholders, complex problem, creative solution.*

**S — Situation**

We'd built a budget bill versioning system — six version types, each with multiple versions, displayed as something like "type-number-version." Drafters found it confusing and wanted it simplified.

**T — Task**

The initial ask was to rework the versioning system itself — drop the version number entirely, one version per type. That meant rewriting significant code across the app, close to a session deadline, with real risk of breaking things.

**A — Action**

I dug into what was actually confusing them. The problem wasn't the data model — it was the display. The document view was cluttered with all those version details. So instead of rearchitecting, I suggested keeping the system as-is but hiding the version number in the document view: same data underneath, cleaner display for the user.

**R — Result**

They agreed. We got a quick win, avoided a risky refactor close to deadline, and the drafters got the clarity they actually needed. Lesson: diagnose what the stakeholder needs (clarity), not just what they ask for (simplification) — sometimes the answer is UI, not code.

**🔥 Likely pushback**

- **Q: What if the data model really had been the problem — how would you have known?** A: I checked their specific complaints against the actual data before proposing anything — each one traced to what was shown on screen, not to a case where two versions were genuinely indistinguishable underneath. If I'd found that case, the UI fix wouldn't have been enough and I'd have said so.
- **Q: Did anyone push back that hiding it in the UI felt like papering over the real issue?** A: Not really — once it was clear the model wasn't changing and this was purely about what's displayed, it was an easy yes given the deadline. If they'd wanted the number gone from the data too, that stayed on the table as later, non-urgent work.

**🗣️ Spoken version (~60-90s), ready to rehearse at this length:**

> *"Early in my Propylon work, we built a budget bill versioning system — six version types, each with multiple versions. Drafters found the naming confusing and wanted it simplified. The initial ask was to rework the versioning system itself, which meant rewriting significant code close to session deadline, with real risk. I dug into what was actually confusing them — turned out the problem wasn't the data model, it was the document display, which was cluttered with version details. So instead of a risky refactor, I suggested hiding the version number in the document view while keeping the underlying system unchanged. They agreed — quick win, no refactor risk, drafters got the clarity they needed. Lesson: diagnose what the stakeholder actually needs, not just what they ask for — sometimes the fix is UI, not architecture."*

### Initiative / improving a process or introducing a technology

**Looking for:** did you spot the problem before someone told you to fix it, and did you justify the investment (why this tool/process, what was the cost of the status quo)? They want evidence you don't just execute tickets. Your DocIntel/Python upskilling angle fits here if you built something with it, not just learned it.

**Story A — process improvement: client bug ticket quality (Montana).**

- **S** — Clients raised bug tickets with just a screenshot and a short description of what's wrong and what they expected. That was rarely enough to act on — we'd need logs, the files where the issue occurred, exact replication steps, full-screen screenshots.
- **T** — Every under-specified ticket meant a clarification round-trip with the client before investigation could even start, delaying fixes. Nobody owned fixing the intake process; I took it on.
- **A** — Defined what an actionable ticket needs (logs, affected files, exact replication steps, full-screen screenshots) and got that baked into how clients raise tickets, so the information arrives up front instead of being chased afterwards.
- **R** — Far less back-and-forth with the client; tickets arrive actionable and investigation starts immediately instead of after a clarification cycle.

**🔥 Likely pushback (Story A)**

- **Q: How did you get clients to actually comply, rather than keep submitting screenshots?** A: It wasn't a technical gate — it needed the support process to actually push back and ask for the missing pieces the first few times before it stuck. Adoption wasn't instant; it held because unactionable tickets got bounced back instead of triaged anyway.
- **Q: Do you have a number on the reduction in round-trips?** A: No hard metric — it was clearly noticeable to the team from fewer "please provide more information" replies, but if pushed for a figure I don't have one. Fair gap in this story.

**Story B — introducing a technology: AI-assisted coding practices (Montana).**

- **S** — The team was adopting AI coding assistants (Copilot / Claude) ad hoc — no shared conventions, so the tools gave verbose, unfocused output and occasionally did damage, like editing VS autogenerated designer code in the VSTO project, which leads to unexpected results.
- **T** — Make the tooling reliable and cheap enough to be worth using, rather than each person burning credits re-explaining context every session.
- **A** — Introduced instruction files (copilot-instructions / claude.md) encoding how the assistant should work: concise by default, verbose only when necessary (caveman rule); never touch VS autogenerated code like designer files; follow our coding styles. Used graphify to generate an index map of the codebase architecture so the assistant navigates instead of re-reading everything, with a rule to prompt me to regenerate it after structural changes.
- **R** — The team burned through their credits much slower, and the assistant's output became safer and more consistent — no more designer-file edits.

**🔥 Likely pushback (Story B)**

- **Q: How do you know the instruction files caused the improvement, rather than the team just getting better with the tools over time?** A: Can't fully separate the two — some of it is a natural learning curve. But the designer-file damage stopped right after we added the explicit "never touch autogenerated code" rule, and that before/after is distinct enough that I'm confident the instruction file did real work, not just general familiarity.
- **Q: What's the actual before/after on credit usage?** A: Observed via usage/billing visibility, not a formal study — no clean chart. Directional result I'm confident in, wouldn't oversell as rigorously measured.

### Prioritising under competing deadlines

**Looking for:** your reasoning method (impact vs effort, risk, who's blocked), and that you communicated tradeoffs upward rather than silently dropping something. A red flag answer is "I just worked more hours" — they want a prioritization framework.

### A difficult decision or tradeoff

**Looking for:** that you can articulate the alternatives you didn't pick and why, own the outcome (including if it was imperfect), and show judgment under uncertainty/incomplete information. This is the one where "and it worked out great" answers feel weakest — a tradeoff with a real cost you accepted is more convincing.

**Story: PDF generation strategy for condensed amendments (Propylon).** Within the same condensed-amendment PDF feature (see Ownership end-to-end) — the choice of *how* to generate the PDF, fidelity vs. simplicity.

- **S/T** — The feature needed to turn a Word document into a PDF containing only selected pages. Two real options: server-side headless conversion (LibreOffice), or routing generation through the VSTO add-in using Word's own print engine.
- **A** — LibreOffice was the obvious architecture: synchronous, no client dependency, fully automatable, batchable, and simpler to build and operate. But these are legal documents, and the existing manual process the team was replacing used Word's own print engine — its pagination is the source of truth. LibreOffice re-flows text differently, so a line wrapping onto a different page could change what a committee is looking at. I judged that risk as unacceptable for a legal artifact and chose the Word-based route instead, even though it was architecturally more expensive.
- **The cost I accepted** — routing through Word made everything downstream harder: the `ms-word:` launch URI can't carry parameters, so I had to smuggle the page selection into the docx as custom XML metadata for the add-in to read on open; generation now depends on the user's machine having Word and the add-in installed; and the whole flow became asynchronous and non-batchable from the server's point of view, which meaningfully increased the scope of the C# side of the spec I had to write.
- **R** — The generated PDFs were pixel-for-pixel products of the same engine the manual process used, so committees and downstream automation saw zero difference, and the feature shipped without a fidelity dispute ever surfacing. The tradeoff was complexity and automatability given up for correctness on a legal document — not a free win.

**🔥 Likely pushback**

- **Q: Did you actually test LibreOffice and observe pagination drift, or was this a prediction?** A: I didn't build a LibreOffice prototype to compare — the call was based on knowing Word's and LibreOffice's rendering engines are different implementations of the same spec, on a document type where the committee treats the Word-produced manual output as ground truth. I judged the risk of even one pagination mismatch as unacceptable rather than proving it would happen; testing it would have de-risked the call further if I'd had the time.
- **Q: Doesn't requiring Word on the user's machine just push the same fragility onto every drafter's desktop?** A: Yes, and I accepted that deliberately. But the environment already assumed every drafter has Word open with the add-in installed for the rest of their workflow — I was extending an existing dependency, not introducing a new one.
- **Q: What if the client had insisted on a fully server-side, no-desktop architecture?** A: Then the answer changes — either accept some pagination risk with LibreOffice and add heavier verification (page-count or diff checks against the Word original), or look at licensed server-side Word automation. I didn't have to make that call here because the desktop-Word dependency was already a given.

### Team collaboration

**Looking for:** a concrete instance of making someone else more effective (unblocking, reviewing, pairing, filling a gap) — not just "we worked well together." They're listening for humility (crediting others) balanced with your specific contribution.

**Story: unblocking the HB2 HTML-to-Word conversion (Montana).** HB2 is a standalone, long-running project — a full drafting process for the appropriations bill (C# drafting, amendments, engrossment, conflict reports) plus a Python API converting the Finance department's HTML budget tables into a Word document. 

**S — Situation**

A colleague building that html→docx conversion hit a wall: the Finance department's tables needed to reproduce the legislature's traditional tab-aligned appropriations layout once opened in Word, not render as an HTML `<table>`. Neither a real `<table>` nor literal tab characters in the source survived Word's import — they got lost or reflowed. He was fully stuck on it.

**T — Task**

Nobody assigned me this — I stepped in on his blocker because it was going to stall the whole conversion pipeline he'd already built.

**A — Action**

I went looking specifically for how Word represents a tab when it parses HTML, rather than guessing. Saving a tab-aligned document as HTML from Word and diffing it against a plain HTML table surfaced Word's own proprietary style hints — Microsoft's `mso-*` CSS dialect that only Word's HTML importer interprets. The specific property was `mso-tab-count`: a style on a `<span>` that tells Word to insert a literal tab character at that point, invisible in a normal browser. I wrapped it into a small reusable element (`get_tab_count_element()`) that dropped straight into the conversion logic he'd already written, wherever a column boundary needed to become a real Word tab instead of a `<td>`.

**R — Result**

Fully unblocked him — that one property became the load-bearing mechanism the entire HB2 table conversion depends on today. HB2 went live in production for the 2025 legislative year, and we were both named and recognized by Propylon's SVP for the work.

*(That colleague is also who referred me to this interview — good context for you, not necessarily a line to say out loud unless it comes up naturally.)*

**🔥 Likely pushback**

- **Q: Why hadn't your colleague found it already — does this undersell his contribution?** A: He'd already built the entire row, cell, and department-header structure — that's the hard part, and it was solid. This was one narrow corner of Word's proprietary HTML dialect neither of us had reason to have hit before; I got there because I specifically went looking for "how does Word represent a tab in HTML," not because his approach was wrong.
- **Q: Was this really collaboration, or you solving his problem solo?** A: I was working his blocker in direct response to being stuck alongside him at that point in the project, and the fix plugged straight into the pipeline he'd already built — it's his code, my missing piece.
- **Q: How do you know `mso-tab-count` was the right fix and not a hack?** A: It's a real, if obscure, part of Word's own HTML/CSS extension set, not a workaround — confirmed against Microsoft's published HTML/CSS extensions reference, and it's been the stable mechanism in production since.

**Alternate (thinner, keep as backup):** providing prompt reviews for a colleague during an important version-upgrade project, putting my own work on hold to do it — solid "made someone else more effective" shape, but no S/T/A/R detail or result behind it yet.

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
- Primary troubleshooting story: the public-member deletion bug (CM-308). Mortgage-protection is backup.
- Finish the customer-similarity story's ending before the interview.
- Keep answers tight — give the signal, not the saga.
