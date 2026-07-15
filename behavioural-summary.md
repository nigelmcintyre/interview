Opening positioning line (use if asked “tell me about yourself” or to bridge CV vs history)

What to say: “Full-stack engineer, about five years — strongest in C#/.NET and financial-services software at Irish Life, plus document-systems work at Propylon across a Python/Django and Postgres backend. FastAPI, RAG, and the agentic side are newer to me, so rather than just claim the keywords, I built DocIntel end-to-end to actually understand it. I’d rather tell you what I built and where it’s rough than oversell it.”

Why it works: referral-backed hire → they value real signal over polished claims. Their spec explicitly wants responsible-AI awareness and knowing limitations, so this honesty reads as strength, not weakness.

Rule to hold yourself to: only say “I built X” for things you can defend under one follow-up.

1. Difficult troubleshooting / debugging

What they’re scoring: systematic method, not luck — hypothesis, isolate variables, logs over guessing. For third-level support specifically: distinguishing symptom from root cause, communicating status while still investigating. Bonus: did you leave the system better.

Your primary story — public member assets deleted on committee removal:

“A client bug ticket reported that removing a public member from an interim committee was wiping that person’s assets, not just their membership on that one committee. I set up a dummy public member on a dummy committee in a test environment so I could reproduce on demand. Reproducing confirmed it — I checked logs to see which endpoint fired on removal, then stepped through both the Aurelia frontend and Django backend to trace the flow. I found the frontend tracked removed public members separately and sent their UUIDs to the backend as removedPublicMembers — and the backend had a special-cased block that ran an unscoped hard delete on the User record for anyone in that list. Every other member removal in the same function only deleted the committee-specific join row. So the bug was a conflation of two different operations — ‘remove from this committee’ had been implemented as ‘delete this person entirely.’ I fixed it by removing that hard-delete path so public members go through the same scoped deletion as everyone else. Ticket resolved, and it removed a live data-loss risk, not just patched a symptom.”

Likely pushback:

	•	“Was that hard-delete special case intentional?” → “I didn’t dig into commit history — priority was closing the live data-loss risk, not archaeology. Best guess: it was written for a different use case and got wired into the wrong flow. Worth a follow-up ticket, not worth blocking the fix.”
	•	“How did you know your fix wouldn’t break something else relying on that cascade?” → “Every other member-type removal in the same function was already doing the scoped deletion — mine was the outlier, not the norm. Matching the existing pattern was lower-risk than the special case ever was.”
	•	“Shouldn’t a unit test have caught this?” → “There wasn’t coverage on that specific path — which is exactly how it shipped. That’s a real gap I’d own, not explain away.”

Backup story (Irish Life): mortgage-protection quote — got a dummy account from the broker-app team, replicated in QA, found the customer wasn’t eligible for the product, told the broker, raised a ticket for an eligibility check to prevent recurrence.

2. Ownership end-to-end

What they’re scoring: did you drive it from ambiguous ask to production, including the unglamorous parts (deployment, monitoring, telling stakeholders it’s done)? A moment where you made a judgment call without waiting for permission.

Your story — tabbed-table HTML export:

“Bill exports render real Word tables into HTML fine, but ‘tabbed tables’ — legislative content built from tab-separated paragraphs, used throughout appropriations bills — collapsed into mangled, misaligned text. No one asked me to fix it — I found the gap while working the export pipeline and owned it end-to-end across the C# Word add-in and the Python document API.

First judgment call: split the work — detection in C#, since the add-in already has the document structure; conversion in Python, keeping it a pure, testable function. Second problem: there’s no dedicated paragraph style marking these as tables, so I had to find a structural rule from real bill samples — rows have two or more interior tab groups, ordinary statute text doesn’t. The highest-stakes call was correctness: these documents carry legal amendment marking — strike and underline — through character styles, and a converter that ignored that would silently delete legally meaningful markup. I mapped those styles to the same markup the existing real-table export already used, rather than inventing something new.

I kept it reversible throughout — behind a flag, falls back to the old behaviour on any API failure, and the toggle-off path had to produce byte-identical output as a regression guard. Shipped — tabbed tables now render as proper HTML tables, visually indistinguishable from real Word tables, with the existing path verified unchanged.”

Likely pushback:

	•	“Why route through the add-in instead of a standalone headless converter?” → “The add-in already has Word’s own view of the document for free — rebuilding that from raw XML risked drifting from what the actual export produces.”
	•	“What happens if the API is slow or down mid-export?” → “Short timeout, and any failure falls back to the pre-existing output. The feature can only fail open, never fail the export itself.”
	•	“That tab-count heuristic sounds fragile — what if a real bill breaks it?” → “It’s tuned against real samples, not a proof, and I know that. A misdetected table surfaces as a per-table warning and falls back locally — not a silently wrong legal document. Fail visibly and locally was deliberate.”

3. Non-technical / cross-functional stakeholders

What they’re scoring: translation skill — explaining a technical constraint in terms a business user cares about. Maps directly to Citco’s fund accountants/clients who use the system but aren’t engineers.

Your story — budget bill versioning display (you’ve already rehearsed this one at length):

“We’d built a budget bill versioning system — six version types, each with multiple versions. Drafters found the naming confusing and wanted it simplified. The initial ask was to rework the versioning system itself, which meant rewriting significant code close to session deadline, with real risk. I dug into what was actually confusing them — turned out the problem wasn’t the data model, it was the document display, which was cluttered with version details. So instead of a risky refactor, I suggested hiding the version number in the document view while keeping the underlying system unchanged. They agreed — quick win, no refactor risk, drafters got the clarity they needed. Lesson: diagnose what the stakeholder actually needs, not just what they ask for — sometimes the fix is UI, not architecture.”

Likely pushback:

	•	“What if the data model really had been the problem — how would you have known?” → “I checked their specific complaints against the actual data before proposing anything — each one traced to what was shown on screen. If I’d found a case where two versions were genuinely indistinguishable underneath, the UI fix wouldn’t have been enough, and I’d have said so.”
	•	“Did anyone push back that this felt like papering over the real issue?” → “Not really — once it was clear the model wasn’t changing and this was purely about display, it was an easy yes given the deadline.”

4. Initiative / introducing a technology or process

What they’re scoring: did you spot the problem before someone told you to fix it, and justify the investment? Evidence you don’t just execute tickets. Your DocIntel angle fits here — if you can point to something you built with it, not just learned.

Story A — client bug ticket quality:

“Clients raised bug tickets with just a screenshot and a short description — rarely enough to act on. Every under-specified ticket meant a clarification round-trip before investigation could even start. Nobody owned fixing the intake process, so I took it on — defined what an actionable ticket needs (logs, affected files, exact replication steps, full-screen screenshots) and got that baked into how clients raise tickets. Result: far less back-and-forth, tickets arrive actionable.”

Likely pushback: “Do you have a number on the reduction?” → “No hard metric — noticeable to the team from fewer ‘please provide more info’ replies, but I don’t have a figure if pushed. Fair gap in this story.”

Story B — AI-assisted coding conventions:

“The team was adopting AI coding assistants ad hoc — no shared conventions, so output was verbose and occasionally damaging, like editing auto-generated designer code in the VSTO project. I introduced instruction files encoding how the assistant should work — concise by default, never touch auto-generated code, follow our coding style — plus a generated architecture index so the assistant navigates instead of re-reading everything. Result: the team burned through credits much slower, and the designer-file damage stopped.”

Likely pushback: “How do you know the instruction files caused the improvement, not just the team getting better with the tools over time?” → “Can’t fully separate the two, but the designer-file damage stopped right after we added that explicit rule — that before/after is distinct enough to be confident the instruction file did real work.”

Matches almost word-for-word with Citco’s “investigate new tech and present for architectural review” ask — worth naming that connection if it comes up.

5. A difficult decision or tradeoff

What they’re scoring: can you articulate the alternatives you didn’t pick and why, own an imperfect outcome, show judgment under uncertainty. “And it worked out great” is the weak version — a real accepted cost is what convinces.

Your story — PDF generation strategy for condensed amendments:

“The feature needed to turn a Word document into a PDF with only selected pages. Two real options: server-side headless conversion with LibreOffice, or routing through the VSTO add-in using Word’s own print engine. LibreOffice was the obvious architecture — synchronous, no client dependency, fully automatable. But these are legal documents, and the process being replaced used Word’s own pagination as the source of truth. LibreOffice re-flows text differently, so a line wrapping onto a different page could change what a committee is looking at. I judged that risk unacceptable and chose the Word-based route instead, even though it was architecturally more expensive — I had to smuggle page selection into the docx as custom XML metadata, generation now depends on the user having Word installed, and the whole flow became asynchronous instead of batchable. Result: the PDFs were pixel-for-pixel products of the same engine the manual process used — zero fidelity dispute. The tradeoff was complexity and automatability given up for correctness on a legal document. Not a free win.”

Likely pushback:

	•	“Did you actually test LibreOffice and observe pagination drift, or was this a prediction?” → “I didn’t build a comparison prototype — the call was based on knowing they’re different rendering implementations of the same spec, on a document type where Word’s output is treated as ground truth. I judged the risk of even one mismatch as unacceptable rather than proving it would happen.”
	•	“Doesn’t requiring Word on the user’s machine just push the fragility onto every drafter’s desktop?” → “Yes, and I accepted that deliberately — but the environment already assumed every drafter has Word open with the add-in for the rest of their workflow. I was extending an existing dependency, not introducing a new one.”

6. Team collaboration

What they’re scoring: a concrete instance of making someone else more effective — not just “we worked well together.” They’re listening for humility (crediting others) balanced with your specific contribution.

Your story — unblocking the HB2 HTML-to-Word conversion:

“A colleague building an HTML-to-Word conversion hit a wall: Finance’s tables needed to reproduce a traditional tab-aligned layout once opened in Word, but neither a real table nor literal tab characters survived Word’s import. He was fully stuck. Nobody assigned me this — I stepped in because it was going to stall the pipeline he’d already built. I went looking for how Word actually represents a tab when parsing HTML — saved a tab-aligned document as HTML from Word and diffed it against a plain HTML table, which surfaced Word’s own proprietary CSS property, mso-tab-count, that tells Word’s importer to insert a literal tab. I wrapped it into a small reusable element that dropped straight into his existing conversion logic. Fully unblocked him — that property became the load-bearing mechanism the whole HB2 table conversion depends on today. It went live for the 2025 legislative year, and we were both recognized by Propylon’s SVP for it.”

Likely pushback:

	•	“Does this undersell his contribution?” → “He’d already built the entire row, cell, and header structure — that’s the hard part. This was one narrow corner of Word’s proprietary HTML dialect neither of us had reason to have hit before.”
	•	“Was this really collaboration, or you solving his problem solo?” → “I was responding directly to his blocker, and the fix plugged straight into the pipeline he’d already built — it’s his code, my missing piece.”

(Side note: that colleague is the one who referred you to Citco — good context for you, not necessarily a line to say unless it comes up naturally.)

Alternate backup: prompt reviews for a colleague during a version-upgrade project, putting your own work on hold — solid shape, thinner detail.

7. Prioritising under competing deadlines

What they’re scoring: your reasoning method (impact vs effort, risk, who’s blocked), and that you communicated tradeoffs upward rather than silently dropping something. Red flag answer: “I just worked more hours.”

Backup story (Irish Life): asked to add a webchat feature mid-project, both due the same deadline — raised it early in stand-up rather than quietly slipping, team split the sprint work, both shipped on time.

8. Learning something new quickly

What they’re scoring: your learning method under time pressure — how you scoped what to learn vs. skip, what resources you used — and proof of application, not just comprehension. A demo beats “I read the docs.”

Your natural answer: DocIntel itself is the proof — you scoped it specifically to close named gaps (FastAPI, RAG, modern React) rather than reading about them abstractly, and you can point to working code as the output.

Backup stories (ready if a primary story gets used up or doesn’t fit)

	•	Performance investigation (Irish Life/OLS) — measured method execution times, found over-fetching, switched endpoints. Doubles as your “slow API” technical answer too.
	•	Initiative + data-driven decision (Infobip/Ansible) — measured dev environment setup time, learned Ansible, took cost/benefit to a retro. Matches Citco’s “investigate new tech, present for review” ask closely.
	•	Ownership + stakeholders (Irish Life/OLS integration) — requirements, API endpoints, deep links, Azure AD SSO, supporting QA. Financial services + API/SOA + auth in one story.
	•	Team player under pressure — covered a colleague’s ticket before their holiday so the sprint deadline held.

⚠️ Unfinished: the customer-similarity bug story (flag a bug near a deadline, risk missing it vs. letting QA catch it) — your notes don’t record the outcome. Write the honest ending before using it; it almost certainly should end with “I flagged it early,” consistent with your own stated principle of never hiding delays.

Strengths / weaknesses / career goals

Strengths: learns new tech fast and is eager to (DocIntel is the proof) · organised, juggles concurrent tasks · communicates well across technical and non-technical people. Back each with a one-line example.

Weakness, framed with a fix: “I like to fully understand why something works before I’m confident with it, so I front-load learning — which is exactly why I built DocIntel rather than just reading about RAG.”

Career goals: deepen technical range into new stacks (Python/FastAPI/AI is exactly that), more building from scratch and extending existing systems, moving toward designing/evaluating architectures.

Questions to ask them

	•	What does the MOS platform stack look like today, and where are you introducing agentic workflows — production or exploratory?
	•	Split between greenfield features vs. enhancing the existing treasury/collateral/trade platforms?
	•	Are you using Bedrock/AgentCore in production yet, or building toward it?
	•	Typical release cycle and third-level-support rota?
	•	Team size/structure — who would I work most closely with, who’s the Technical Lead I’d report to?
	•	What does success in the first six months look like?

Quick cheat sheet

	•	Lead with financial services + strong debugging/ownership; frame the AI stack as the gap you built DocIntel to close.
	•	Never claim past what you’ve built and understand — “I’m building X” is a strong, honest answer.
	•	Primary troubleshooting story: public-member deletion bug. Mortgage-protection is backup.
	•	Finish the customer-similarity story’s ending before the interview.
	•	Keep answers tight — signal, not saga.