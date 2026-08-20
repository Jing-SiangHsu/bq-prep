# Modular Cover Letter Template

Four-paragraph structure: hook, why-you, why-this-company, close. Under one page, always. Every paragraph earns its place, if a sentence could be cut without losing anything, cut it.

**Not for Google** — Google's application process doesn't accept cover letters (confirmed this session). This is for the smaller companies/startups you apply to alongside it, where a tailored letter meaningfully moves callback rates.

**What's reusable vs. what must be researched fresh every time:**
- Paragraph 2 (why you) draws from the story bank below, reuse these near-verbatim.
- Paragraphs 1 and 3 (hook, why this company) **cannot** be pre-written, they only work if they reference something real and specific about that company. Budget 10-15 minutes of research per application for these two.

---

## Paragraph 1 — Hook

Why this role, why now, why you noticed it. Never open with "I am writing to express my interest in the position of..." — it tells the reader nothing and wastes the one sentence they're most likely to actually read.

Lead with whatever is most specific and true: a direct connection (you use their product, someone referred you, you've followed their work), or a concrete line from the JD that maps to something you've actually built.

*Example, tailored to your background:*
> "Your listing mentioned rebuilding \[X\] to handle real concurrent load, that's almost exactly the problem I spent the last year solving as the sole engineer on a concurrent test-editing platform used by 30 engineers at Intrising Networks."

---

## Paragraph 2 — Why You (story bank)

Pick **one** of these, not multiple. Each is a compressed CART story (Context/Action/Result/Takeaway in 3-5 sentences), matching cover-letter length, not interview-answer depth. The full versions of these same stories live in `interview_stories_cart.md`/`star.md` for BQ prep, use those when the format allows more room. Match to what the JD actually emphasizes, don't guess.

### A — Ownership under ambiguity
*Use when the JD emphasizes ownership, independence, ambiguity, or 0-to-1 work.*

As the sole engineer on AutoPVT, a concurrent test-editing platform used daily by around 30 engineers at Intrising Networks, I was responsible for designing its entire concurrency model from scratch. I mapped out how edits across the platform's four-step workflow could collide, then closed a gap where one page's edit could silently destroy another tester's results, making the system reject the conflict explicitly instead. It ran in daily use with no data-corruption incidents for as long as I was there. That kind of ownership means deciding what's correct when no one else has defined it yet.

### B — Conflict resolution / earning trust
*Use when the JD emphasizes collaboration, cross-team work, or "earn trust"-style values.*

As the de-facto lead on that same platform, I once had to raise a missing API with a colleague who initially denied responsibility, despite clear evidence he'd agreed to build it. Rather than push the disagreement, I re-explained what was needed, asked him to rebuild it, and shared the evidence privately with my manager so the record stayed clear without escalating the conflict. He rebuilt it, the deployment shipped, and our working relationship stayed intact. Trust isn't built by winning the argument, it's built by the other person seeing that you didn't need to.

### C — Accountability / ownership of mistakes
*Use when the JD emphasizes reliability, production ownership, or "bias for action"-style values.*

While refactoring a set of internal cloud functions at Intrising Networks, I added a required parameter to one that had quietly been in production for years, without considering who depended on it, and a customer-facing tool broke a week later. Because I'd started keeping a changelog of exactly what I changed and when, I traced the cause immediately, made the parameter optional, and shipped a fix the same day. I didn't wait to be told it was my change that broke it, owning that immediately is the part of this I'd repeat every time.

### D — Self-directed initiative in AI engineering
*Use when the JD is AI/ML-specific, or when you want to address "why an internship" indirectly by showing genuine self-motivated interest rather than a resume line.*

Outside of my full-time work, I spent two months building an ad-recommendation system on my own initiative, specifically to go deeper into AI engineering ahead of my Master's at UCLA. Beyond the core Kafka-based data pipeline and the LLM agents that review campaigns and onboard users, I red-teamed those agents myself, found real ways to manipulate them, and built a deterministic backstop where a better prompt wasn't enough. No one assigned me this project or this level of scrutiny; I wanted to understand where LLM systems actually break before I'm responsible for one that matters.

---

## Paragraph 3 — Why This Company (research required, every time)

Show you did the homework. Reference their actual product, mission, a recent launch, or something specific about the team you're applying to, then say why it matters to *you* personally, not generic flattery ("I admire your innovative approach" counts for nothing).

Checklist before writing this paragraph:
- [ ] Read the company's own product/blog/changelog, not just the JD
- [ ] Find one specific, recent, true thing (a launch, a technical choice, a stated problem they're solving)
- [ ] Connect it to a real reason it matters to you, not just that it exists

---

## Paragraph 4 — Close

Confident, not desperate. A clear, specific call to action, not a vague "I look forward to hearing from you."

*Template:*
> I'd welcome the chance to discuss how my experience in **\[echo one trait from paragraph 2\]** could support **\[ORGANIZATION\]**'s work on **\[something specific from paragraph 3\]**. I'm available for a conversation at your convenience.

Jing-Siang Hsu

---

## Tone Matching

Calibrate to the company type before writing paragraph 1 and 3:

| Company type | Tone | Example phrasing |
|---|---|---|
| Startup / tech | Conversational, direct | "I've spent the last year building exactly this kind of thing" |
| Corporate / enterprise | Professional, measured | "My experience in distributed systems aligns closely with your stated objectives" |
| AI-native / research-driven | Technical, precise, low on adjectives | State the finding, not the excitement about the finding |

Most companies you'll apply to alongside Google fall into the first two columns.

---

## Common Mistakes

- **Rehashing the resume.** The letter adds context and character, it does not repeat bullet points. Every paragraph above deliberately covers something a resume bullet can't (judgment, conflict, a mistake, motivation), not the technical mechanism of what was built.
- **Generic openings.** "I am excited to apply for..." tells the reader nothing.
- **No company reference.** If the letter could go to 50 companies unchanged, it's too generic, this is the exact failure mode the data on tailored vs. generic cover letters is about.
- **Underselling or overselling.** State what you've done, factually. No "I'm the perfect candidate," no "I know I don't have much experience but..."
- **Burying the lead.** If there's a direct connection (you use their product, a referral, deep relevant experience), say it in the first line, not the third paragraph.

---

## Special Circumstance: Overqualified

**This applies to you directly.** Every reviewer in the 5-agent resume panel independently flagged the same thing: your professional experience (sole ownership of a platform serving 30 engineers, a security-compliance workstream, a full-time title, not "intern") reads senior for an internship application, and every one of them said the first question in a screen would be "why are you applying for an internship?"

The framework's advice: **name the real reason directly**, don't dance around it. Whatever it actually is, tied to your Master's program timeline, wanting focused AI-specific depth before going full-time, visa/program timing, pick the true one and state it plainly in a sentence, ideally in paragraph 1 or woven into paragraph 2's takeaway line. Do not leave it unaddressed and hope the reader doesn't notice, per the panel's unanimous finding, they will notice.

*Draft options, adapt to whichever is actually true for you:*
> "After a year owning production systems independently, I'm looking for a structured, AI-focused internship specifically to build depth in an area I haven't had the chance to specialize in yet, ahead of finishing my Master's at UCLA."

> "This internship lines up with my UCLA program's schedule, and I wanted to spend it somewhere I could go deep on AI engineering specifically, rather than the generalist full-stack work I've been doing."

Pick one, edit it to be true, and use it. Don't skip this.
