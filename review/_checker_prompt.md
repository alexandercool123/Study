# Per-lecture exam fact-checker (focused prompt)

Goal of this pass: for ONE lecture, decide whether anything **important for the oral exam**
is **wrong**, **missing**, or **named-but-not-explained**. Importance is judged through the
examiner's lens — the 9 known exam questions and their a/b/c sub-parts. This is NOT an
exhaustive audit of every sentence, and it is NOT a style/citation-formatting check.

The exam's own grading lens (the bar to hold answers to):
  "Focus is on understanding the principles and mechanisms and the purpose of protocols, technologies"
  "Pay extra attention to central architectural figures, which you might be asked to explain"
So: a student must be able to explain *how something works and why it exists*, and walk an
examiner through the *central diagrams/flows*. Spec-trivia (exact field names, byte layouts)
matters ONLY when getting it wrong would embarrass the student in front of the examiner.

---
## Orchestrator (do once, then spawn one checker per lecture)
For each lecture N (1..11), give the checker three things, substituted into PART B below:
  - {N}
  - {FOLDER}        the lecture's source folder, e.g. "/workspace/05 - 2026-03-10 - Resources, Policies and Identity Management"
  - {EXAMQS}        the exam question(s) this lecture owns + any it cross-supports, copied
                    VERBATIM from /workspace/study/_map.json (it already holds subParts a/b/c
                    word-for-word and the crossRefs). Copy them mechanically — do not retype.
Spawn checkers in parallel / background. Collect the one-line summaries. A coverage "gap" is
only real if it is ALSO absent from the crossRef lectures — reconcile that yourself before
acting on any gap.

---
## PART B — CHECKER PROMPT (read-only; substitute {N}, {FOLDER}, {EXAMQS})

You are fact-checking ONE lecture of an IAM oral-exam study site. You do NOT edit any JSON and
you do NOT rebuild anything. You read the sources, judge the existing Q&A, and WRITE one findings
file. You return a 3-line summary.

LECTURE: {N}
FOLDER:  {FOLDER}
Q&A UNDER CHECK: /workspace/study/questions/lecture_{N}.json
THIS LECTURE OWNS / SUPPORTS THESE EXAM SUB-PARTS:
{EXAMQS}

### Step 1 — Read in full (state line counts; no skimming)
  - The exam sub-parts above (this is what "important" means).
  - lecture_{N}.json — every drill and every exercise.
  - EVERY .txt in {FOLDER} (slide deck + all readings). Pages are marked "===== PAGE n =====".
    These are the lecture's own authority for what was actually taught.

### Step 2 — Judge each drill/exercise on THREE things only (in priority order)

**A. WRONG (highest severity).** Is any claim factually incorrect? Two kinds, label which:
   - `wrong-vs-source`: contradicts this lecture's .txt (e.g. invents a step, misnames a
     component, states a number the slides don't support). Quote the source line that contradicts it.
   - `wrong-vs-reality`: the claim is a *core protocol/standard fact* that is simply incorrect
     in the real world — which usually means the SLIDE itself is OCR-corrupted or outdated, not
     just the drill. Flag these for the high-value items where an examiner would notice:
     OAuth2 grant types & endpoints & auth-code flow order; OIDC tokens/claims/flows; SAML
     IdP/SP assertion exchange; Kerberos AS/TGS/service exchange; FIDO2/WebAuthn
     registration vs authentication ceremony; JWT registered claims & structure; XACML
     PEP/PDP/PAP/PIP; the 7 Laws of Identity (count + names); MitID / EU Digital Identity
     Wallet specifics (these go stale fastest — check freshness). For wrong-vs-reality you MAY
     use authoritative external knowledge, but say so explicitly and keep it in its own field —
     never silently "correct" the source. If unsure, say `uncertain`, don't assert.

**B. MISSING (coverage).** Reading the owned sub-parts as a checklist: is there an important
   concept the examiner would expect that NO drill delivers? List it. "Important" = it appears
   in the sub-part text or is a central mechanism/figure for it. Do NOT list nice-to-haves.

**C. INCOMPLETE SET (the main trap).** If a question implies a SET — "the 7 Laws", "the OAuth
   roles/grants/endpoints", "the XACML components", "the Kerberos exchanges" — the answer must
   deliver the WHOLE set at a depth the student could speak to, not name-and-skip. Worked example:
   a drill that names all 7 Laws of Identity but explains only 2 = INCOMPLETE — if the examiner
   asks about law #4 the student has nothing. Fix = explain every member at one-line depth, using
   the lecture's own source text.

Ignore unless it actually misleads a student: citation-string formatting, keyBeats wording,
paraphrase overlap, ordering of nice-to-have detail. This pass is about substance, not polish.

### Guardrails (don't hallucinate while checking)
  - OCR TRAP: cited quotes are often line-wrapped and use curly quotes. Before deciding a claim
    is unsupported, normalize whitespace/quotes and READ ±15 lines around the relevant page —
    a wrapped quote is PRESENT, not missing. Never declare `wrong-vs-source` off a raw grep.
  - Every WRONG/MISSING/INCOMPLETE finding MUST carry evidence: a verbatim source quote with
    file + page/line, OR an explicit "absent from <file> pages X–Y, which I read to the end."
  - For `wrong-vs-reality`, name the external standard you're relying on. Keep it separate from
    source grounding so the human can see which is which.
  - You are read-only on the JSON. Propose changes; do not apply them. A proposed fix may only
    use this lecture's source text (for wrong-vs-source / missing / incomplete) or a clearly
    cited external standard (for wrong-vs-reality).

### Write exactly one file
/workspace/study/review/lecture_{N}.review.json
{
  "lecture": {N},
  "checked": <drills+exercises read>,
  "findings": [
    {"id":"L{N}-...", "severity":"wrong|missing|incomplete",
     "kind":"wrong-vs-source|wrong-vs-reality|missing|incomplete",
     "what":"<the problem in one sentence>",
     "evidence":"<verbatim source quote + file.txt PAGE n/line, OR 'absent from file pages X–Y (read)', OR external standard named>",
     "fix":"<concrete change grounded in source or named standard>"}
    // ONE entry per problem found. Clean drills need no entry.
  ],
  "coverageVerdict":"<one line: do the drills collectively cover all owned sub-parts? if not, what's the most exam-critical gap?>",
  "_sources_read": ["file.txt:linecount", ...]
}

Then RETURN ONLY:
  "L{N}: <wrong> wrong / <missing> missing / <incomplete> incomplete — review.json written. Most exam-critical issue: <one phrase>."
