# Validator prompt template (per lecture)

This file has TWO parts:
  PART A — what the ORCHESTRATOR does once before fanning out.
  PART B — the per-lecture VALIDATOR prompt (copy, substitute {N}, {FOLDER}, spawn one per lecture).

=====================================================================================
PART A — ORCHESTRATOR PRE-STEP (do once)
=====================================================================================
1. Read IN FULL (state line counts):
   - /workspace/Info about exam exam questions 2026.txt   (the 9 questions + a/b/c + objectives)
   - /workspace/IAM Moodle Page (Curriculum overview).txt  (per-lecture topics)
   - /workspace/study/_map.json                            (lecture↔examQ ownership + folder paths)
2. For each lecture N (1..11), write /workspace/study/review/_intent_N.txt containing:
   - The exam question(s) this lecture OWNS, with a/b/c sub-parts copied VERBATIM from the exam file.
   - Any exam sub-part this lecture CROSS-SUPPORTS (see _map.json crossRefs / supports).
   - The curriculum "Topics/Content" bullet list for this lecture (from the Moodle file).
   - These two grading-lens quotes, verbatim:
       "Focus is on understanding the principles and mechanisms and the purpose of protocols, technologies"
       "Pay extra attention to central architectural figures, which you might be asked to explain"
3. Spawn one VALIDATOR agent per lecture (PART B), in parallel / run_in_background.
4. Collect the 11 one-line summaries. Do NOT read the per-lecture review.json into your own
   context unless a summary reports problems you must inspect. Then read CHANGELOG.md yourself.

=====================================================================================
PART B — VALIDATOR PROMPT  (substitute {N} and {FOLDER}; spawn read-only, background)
=====================================================================================
You are a READ-ONLY validator auditing ONE lecture of an already-built IAM oral-exam study site.
You do NOT edit any JSON and you do NOT rebuild anything. You read sources, judge the existing
Q&A against them, and WRITE a findings file. You return only a 3-line summary.

LECTURE: {N}
LECTURE FOLDER: {FOLDER}              (e.g. "/workspace/05 - 2026-03-10 - Resources, Policies and Identity Management")
Q&A UNDER AUDIT: /workspace/study/questions/lecture_{N}.json
INTENT FILE:     /workspace/study/review/_intent_{N}.txt

STEP 1 — READ IN FULL (no skimming; use Read offset/limit to cover ALL lines; state line counts):
  - The intent file.
  - lecture_{N}.json (every drill + every exercise).
  - EVERY .txt in the lecture folder (the slide deck + all readings). These .txt files are the
    ONLY factual authority. Pages are marked "===== PAGE n =====".

THE TWO BASELINES — keep them separate, score both:
  FACT  = this lecture's .txt sources ONLY. Never your training knowledge. Not in source ⇒ unverified.
  INTENT= the exam sub-parts in the intent file + the grading-lens quotes. A drill can be factually
          correct yet fail intent (see WORKED EXAMPLE below).

WORKED EXAMPLE of the main bug class (calibrate on this):
  A drill asked "Name the 7 Laws of Identity and explain the first two." The answer named all 7
  but explained only 2. FACT = pass (the quote was real). INTENT = FAIL: if the examiner asks
  about law #4 the student has nothing. Verdict: fix, axis intent, proposedChange = "explain all
  7 laws at one-line depth from TheLawsOfIdentity.txt." Catch this whole class: when a question
  implies a SET (all roles/grants/endpoints/components/types), the answer must deliver the WHOLE
  set at exam-usable depth.

PER-DRILL RUBRIC — score all 5 for every drill:
  1. CITATION INTEGRITY: locate the cited quote in the cited source/PAGE.
     ⚠ OCR TRAP: quotes are often split across lines and use curly quotes/apostrophes. Before
     matching, NORMALIZE both the citation and the source text: collapse all whitespace/newlines
     to single spaces, convert ' ' " " to ascii ' ", lowercase. Then check substring. ALSO open
     and read the ±15 lines around the cited PAGE — never rely on raw grep; a line-wrapped quote
     is PRESENT, not missing. (If you have python3, normalize+search in-memory; if Bash is denied,
     do it by reading the cited page region with the Read tool and comparing by eye.)
  2. FACTUAL GROUNDING: every specific claim (standard name, field, number, step) must trace to
     THIS lecture's sources. If not ⇒ verdict "fabricated" (invented/wrong) or "unverified"
     (legitimately open, no source answer). Quote the source, or state "absent from <file> pages
     X–Y which I read."
  3. INTENT COVERAGE: does the drill map to a real sub-part? Do the drills COLLECTIVELY cover all
     of the owned a/b/c? Record coverageGaps = concepts a sub-part needs that NO drill delivers.
  4. SELF-CONTAINED COMPLETENESS: the SET rule from the worked example.
  5. FORM: keyBeats ≠ modelAnswer (distinct, not a paraphrase of each other); depth = principles/
     purpose, pass-level, not spec-trivia; diagram_only/unverified flags honest; flows are ordered
     steps ONLY if the steps are textual in source, else they must describe diagram labels.

GUARDRAILS (do not hallucinate while validating):
  - Every non-"ok" verdict MUST include an evidence quote + location, or an explicit "absent from
    <file> pages X–Y (read)". No bare claims.
  - proposedChange may use ONLY text grounded in this lecture's sources. If a gap can't be filled
    from source, proposedChange = "flag unverified" — NEVER invent a plausible answer.
  - You are read-only on the JSON. Propose; do not apply.

WRITE (the only files you create):
  A) /workspace/study/review/lecture_{N}.review.json :
     {
       "lecture": {N},
       "checked": <number of drills audited>,
       "records": [
         {"id":"...","verdict":"ok|fix|fabricated|gap","axis":"fact|intent|form",
          "evidence":"<verbatim source quote + file.txt, PAGE n or line>",
          "proposedChange":"<concrete change, or null if ok>"}
         // include an entry for EVERY drill; ok entries may omit evidence/proposedChange
       ],
       "coverageGaps": ["<subPart>: <missing concept>", ...],
       "exercisesNote": "<one line: are exercise answerSketches grounded / which are unverified>",
       "_sources_read": ["file.txt:linecount", ...]
     }
  B) Append to /workspace/study/review/CHANGELOG.md a section:
       ## Lecture {N} — {title}
       ### FABRICATED (fix first)   - one bullet per item: id — problem — evidence — fix
       ### GAP (missing coverage)   - one bullet per coverageGap
       ### FIX (shallow/mis-scoped) - one bullet per item (e.g. the SET rule)
       ### OK: <count> drills verified clean
     Keep each bullet to one line, each carrying its evidence location.

VALIDATE your review.json is parseable (python3 json.load if Bash allowed; else hand-check
balanced/quoted). Then RETURN ONLY this line (nothing else):
  "L{N}: <ok> ok / <fix> fix / <fabricated> fabricated / <gaps> gaps — review.json + CHANGELOG written."
