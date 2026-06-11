# IAM Study-Site Validation Pass — Plan & Playbook

Goal: independently re-verify every Q&A item in `study/questions/lecture_*.json` against
(a) the lecture's own OCR'd source `.txt` files and (b) the teacher's intent (the 9 exam
questions + curriculum objectives). Produce a per-lecture changelog of fixes WITHOUT blowing
up the orchestrator's context window. This is a SEPARATE, read-only audit of an already-built
site — it does not rebuild anything.

## Where things live
- Course-level intent sources (read by ORCHESTRATOR only, once):
  - `/workspace/Info about exam exam questions 2026.txt`  (the 9 questions + a/b/c sub-parts + learning objectives) — THE intent spec
  - `/workspace/IAM Moodle Page (Curriculum overview).txt`  (per-lecture topics & reading list)
- The built artifacts under audit: `/workspace/study/questions/lecture_1.json` .. `lecture_11.json`
- The lecture↔exam map: `/workspace/study/_map.json`  (which exam Q each lecture owns; folder paths)
- Lecture source folders: `/workspace/01 - ...` .. `/workspace/11 - ...` (each holds the slide .txt + readings)
- Outputs of THIS pass: `/workspace/study/review/` (created already)

## The two baselines — never blend them
1. FACT baseline = the lecture's own OCR `.txt` files. ONLY these. Never the model's training
   knowledge, never "what's generally true." Not in source ⇒ `unverified`, not "correct anyway."
2. INTENT baseline = the 9 exam questions + their a/b/c sub-parts + curriculum objectives. The
   exam doc's own grading lens (quote it to agents):
   - "Focus is on understanding the principles and mechanisms and the purpose of protocols, technologies"
   - "Pay extra attention to central architectural figures, which you might be asked to explain"
   A drill can be factually perfect yet fail INTENT (e.g. drill L2-2a-5 named the 7 Laws of
   Identity but only explained 2 — useless when the examiner asks about law #4). That is the
   primary class of bug this pass exists to catch. Score BOTH axes.

## Topology (this is how context stays bounded)
- ORCHESTRATOR: reads the 2 course-level intent files + `_map.json` ONCE. For each lecture N,
  writes `study/review/_intent_N.txt` = the verbatim exam sub-part(s) that lecture owns
  (and any it cross-supports), the curriculum topic list for that lecture, and the 2 grading-lens
  quotes above. Then fans out ONE validator agent per lecture (parallel, run in background).
- VALIDATOR agent N: gets ONLY its folder path, its `lecture_N.json`, and `_intent_N.txt`.
  Never sees other lectures. Context cap = one deck's worth (same as the build). READ-ONLY:
  it does NOT edit the JSON. It WRITES findings to disk and RETURNS only a 3-line summary.
- The orchestrator therefore ingests 11 tiny summaries, never 205 drills. That is the anti-blowup.
- DETECT (pass 1, this plan) is separate from APPLY (pass 2, below). Never let a validator
  rewrite content — that silently reintroduces hallucinations via regeneration.

## Per-drill rubric (the validator scores all 5 for every drill)
1. CITATION INTEGRITY — find the cited quote in the cited source file/PAGE.
   ⚠ OCR TRAP: quotes are often split across physical lines and use curly quotes. NORMALIZE both
   the citation and the source (collapse all whitespace/newlines to single spaces, straighten
   ' ' " " to ascii, lowercase) BEFORE substring-matching. And READ the ±15 lines around the
   cited PAGE — do not trust a raw `grep`; a line-wrapped quote is PRESENT, not missing.
2. FACTUAL GROUNDING — every specific claim (standard name, field, number, step) traceable to
   THIS lecture's sources. If not ⇒ verdict `fabricated` (wrong/invented) or `unverified` (open
   prompt, no source answer). Quote the source, or state "absent from <file> pages X–Y (read)."
3. INTENT COVERAGE — does the drill map to a real sub-part? Do the drills COLLECTIVELY cover all
   of a/b/c? List `coverageGaps` (concepts the sub-part needs that no drill delivers).
4. SELF-CONTAINED COMPLETENESS (the L2 catch) — if a question implies a SET ("the 7 Laws", "the
   XACML components", "OAuth roles/grants/endpoints"), the answer must deliver the WHOLE set at
   exam-usable depth, not name-and-skip. The obvious examiner follow-up must already be answered.
5. FORM — keyBeats ≠ modelAnswer (distinct, not paraphrase); depth = principles/purpose, not
   spec-trivia; `diagram_only`/`unverified` flags are honest (esp. flows: ordered steps only if
   textual in source, else describe diagram labels).

## Validator output (written to disk — survives context death)
- `study/review/lecture_N.review.json`:
  {
    "lecture": N,
    "checked": <int>,
    "records": [
      {"id":"L2-2a-5","verdict":"ok|fix|fabricated|gap","axis":"fact|intent|form",
       "evidence":"<verbatim source quote + file.txt, PAGE/line>",
       "proposedChange":"<what to change; or null if ok>"}
    ],
    "coverageGaps": ["<sub-part>: <concept missing across all drills>", ...],
    "_sources_read": ["file.txt:linecount", ...]
  }
- Append to `study/review/CHANGELOG.md`, grouped by severity, one evidence-bearing line each.
- RETURN to orchestrator ONLY: "L{N}: {ok} ok / {fix} fix / {fabricated} fabricated / {gap} gaps. file written."

## Guardrails so the validators don't hallucinate too
- Every non-`ok` verdict MUST quote the source with a location, or explicitly say the text is
  absent from named pages it actually read. No bare assertions.
- "Fixing from knowledge" is FORBIDDEN. A `proposedChange` may only use text grounded in this
  lecture's sources. If a gap can't be filled from source, the proposal is "flag unverified",
  NOT "write a plausible answer."
- Pass 1 is strictly read-only on the JSON.

## Pass 2 — APPLY (run only after you read CHANGELOG.md)
- Option A (human-gated): you skim CHANGELOG.md, then tell an agent to apply specific records.
- Option B (agentic): one apply-agent per lecture edits ONLY the drills named in that lecture's
  review.json, making ONLY the `proposedChange` (no free regeneration), then re-runs the 5-check
  rubric on the touched drills and writes `lecture_N.applied.json`. Re-run the central JSON
  validator (below) afterward.
- The site needs no rebuild; it loads whatever JSON is on disk. Just re-validate parse + anchors.

## Central re-validation (orchestrator, after apply) — same as original Phase 3
- Parse all 11 JSON; confirm all 9 exam questions still anchored; no duplicate drill IDs;
  0 cases of keyBeats==modelAnswer; print drills/exercises per lecture; list remaining unverified.

## Known starting signal
- `L2-2a-5` (lecture_2.json): factually correct (Cameron quote verified at TheLawsOfIdentity.txt
  lines 605–609) but INTENT-incomplete — names 7 Laws, explains only 2. Fix: explain all 7 at
  one-line depth (each law's defining sentence is in TheLawsOfIdentity.txt: User Control/Consent
  ~line 603, Minimal Disclosure ~659, Justifiable Parties ~735, Directed Identity ~818, Pluralism
  ~940, Human Integration ~1026, Consistent Experience ~1132). Use these as the worked example
  in the validator prompt so agents calibrate on the right kind of defect.
