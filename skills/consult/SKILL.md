---
description: Run a structured rare disease differential diagnosis consultation using Claude with live HPO and PubMed lookup tools. Use when the user describes undiagnosed symptoms, is going through a diagnostic odyssey, asks "what could this be?" about a hard-to-explain symptom cluster, or wants to prepare for a specialist visit about hard-to-diagnose issues. Gathers comprehensive clinical information through conversation, then performs grounded differential reasoning by extracting clinical features, searching the Human Phenotype Ontology, querying PubMed for related cases, and synthesizing a top-5 differential with both patient-facing and doctor-facing detail. Does not provide medical advice or replace physician judgment.
---

# Rare Disease Diagnostic Consultation

## What this skill is and isn't

**What it is:** a structured research workflow that uses Claude plus publicly available medical knowledge tools (HPO, PubMed, general web) to produce a ranked list of rare disease possibilities the user may want to discuss with their physician. A single Claude agent works the case end-to-end with full tool access — extracting clinical features, mapping them to Human Phenotype Ontology terms, looking up candidate diseases by phenotype, querying PubMed for similar published cases, and synthesizing a grounded differential. On a held-out benchmark of 371 post-cutoff rare-disease cases run under contamination filtering (source-paper retrieval at inference time blocked), this workflow scored **76.0% Top-1 / 83.8% Top-5** (Wilson 95% CI for Top-1: [71.4%, 80.1%]), a statistically significant improvement over Claude without tools (+21.1pp, McNemar p<0.0001) and over vanilla Claude (+28.3pp, p<0.0001). Honest interpretation: mid-to-high 70s on this cohort, not exactly 76.0% — single-shot agent benchmarks at this scale carry substantial run-to-run variance, and the Wilson CI under-states the true uncertainty for agent runs.

**What it is NOT:**
- A diagnosis. Only a physician can diagnose.
- Medical advice. Nothing in the output should be acted on without physician review.
- A substitute for urgent care. If the user describes signs of a medical emergency (chest pain, stroke symptoms, severe acute distress, suicidal ideation), stop the consult immediately and recommend they call emergency services.
- A treatment recommender. This skill never recommends starting, stopping, or changing medications.

**Framing:** The output is designed to be a "second opinion research assistant" the user brings TO their doctor, not one they use INSTEAD of a doctor.

---

## Workflow overview

The consultation runs in four phases. Follow them in order. Do not skip phases.

1. **Intake** — conversational information gathering with systematic gap detection
2. **Synthesis** — compose a structured clinical summary from the intake, confirm with the user
3. **Diagnostic analysis** — extract clinical features, map to HPO, look up candidate diseases, query PubMed for similar cases, synthesize a grounded top-5
4. **Presentation** — format the results as a single unified document (patient-scannable summary + doctor-verifiable detail)

---

## PHASE 1 — Intake

### Step 1.1 — Safety check first

Before anything else, scan the user's opening message for red flags that require immediate escalation rather than research:

- Chest pain (especially crushing, radiating, with shortness of breath)
- Signs of stroke (sudden weakness, facial droop, speech trouble, sudden severe headache)
- Severe acute dyspnea or cyanosis
- Active bleeding that won't stop
- Signs of sepsis (high fever with confusion, rapid deterioration)
- Acute abdomen (severe, sudden, localizing abdominal pain with guarding)
- Active suicidal ideation with plan
- Pediatric red flags: non-blanching rash, lethargy not arousable, seizure, infant fever

If any of these are present, **stop the consult immediately** and output:

> "Some of what you're describing sounds like it needs urgent medical evaluation rather than research. I'm going to stop this consult and recommend you call [911 / your local emergency number / Poison Control / a suicide hotline]. We can come back to rare-disease research once you're safe."

Do NOT continue into the diagnostic workflow in these cases.

### Step 1.2 — Open-ended narrative

Start with an open-ended question that gives the user room to tell their story. Use wording that signals you're there to research WITH them, not evaluate them:

> "I'd like to help research what might be going on. To do that well, I need to understand your full picture — not just the main symptom, but the timeline, what's been tried, and what's been ruled out. Start wherever you want: tell me what's been happening."

Read their response carefully. Do NOT interrupt their narrative with clarifying questions mid-flow. Let them finish.

### Step 1.3 — Systematic gap check

After the narrative, silently assess what you have against the full intake checklist below. Then ask ONE consolidated follow-up message that probes only the gaps, grouped naturally.

**Full intake checklist** (what you're trying to have by the end of Phase 1):

**Required minimum:**
- [ ] Age (approximate is OK)
- [ ] Biological sex
- [ ] At least two distinct symptoms (rare diseases are almost always multi-symptom)
- [ ] Timeline: when did this start, how has it evolved
- [ ] What has been done so far — any doctors seen, tests run

**Strongly preferred:**
- [ ] Ethnicity or ancestry (some rare diseases are population-linked)
- [ ] Family history — any relatives with similar symptoms or diagnosed rare conditions
- [ ] Past medical history — other diagnoses, surgeries, relevant childhood issues
- [ ] Current medications
- [ ] What's been formally ruled out
- [ ] Trigger or pattern — what makes it worse, better, episodic vs. constant
- [ ] Functional impact — what it prevents them from doing

**Nice to have:**
- [ ] Specific lab or imaging results (not just "my bloodwork")
- [ ] Any unusual physical features (skin findings, facial features, joint hypermobility, height/growth)
- [ ] Developmental history (for younger patients or patients with childhood-onset symptoms)
- [ ] Environmental exposures — occupation, travel, known toxin exposure
- [ ] Diet or nutritional context if symptoms are GI/metabolic

**How to phrase the gap check** — don't list the checklist back at the user. Group the gaps naturally:

> "Thanks, that gives me a lot to work with. A few things would help sharpen the research:
>
> 1. I didn't catch [age and/or sex]. And — is there anything in your family history worth noting? Parents, siblings, or extended family with anything similar, or with any condition that has a genetic component?
>
> 2. [Any specific test results or imaging] — if you have actual lab values or reports, paste what you've got. 'Normal bloodwork' is less useful than 'ANA negative, CBC within range, ESR 42'. But just tell me what you remember.
>
> 3. [Rule-outs] — are there diagnoses that have already been formally excluded? That helps me not waste time on them.
>
> Anything else you think is relevant, even if it seems unconnected — mention it. Rare diseases often present with weird constellations."

If the user says "I don't know" or "my doctor didn't tell me" for anything, that's fine — record it as "unknown" in the synthesis. Don't badger.

### Step 1.4 — Gate check before Phase 2

After the gap-check round, verify you have at least the **required minimum** fields. If you don't:

- If the user is clearly struggling to provide more info, proceed with what you have and note the limitation explicitly in Phase 2.
- If the user seems able to provide more but hasn't, one more targeted follow-up is OK. Don't make it more than two total follow-up rounds.

Rare disease diagnosis needs enough signal to work. If you have only one symptom and no timeline and no history, the analysis will produce nothing useful. In that case, say so:

> "I want to be straight with you: the strongest research I can do is with multi-system symptoms, timeline, and some history. Right now I have [X], which is thin for what rare disease research needs. I can still try, but the results will probably be very broad. Would you rather add more context first, or go ahead with what we have?"

Respect the user's choice.

---

## PHASE 2 — Synthesis

### Step 2.1 — Write the clinical summary

Compose a structured, physician-readable case summary from the intake. Format it like a brief clinical vignette. Target 150-400 words.

Use this structure:

```
**Clinical Summary**

- **Demographics:** [age] [sex], [ethnicity if known]
- **Chief complaint:** [1-2 sentences in the patient's own framing]
- **History of present illness:** [timeline paragraph — onset, evolution, current severity]
- **Associated symptoms:** [list of secondary symptoms, with any known pattern]
- **Past medical history:** [relevant diagnoses; "none reported" is OK]
- **Family history:** [relevant items; "unremarkable" or "unknown" if applicable]
- **Medications:** [current; "none reported" if applicable]
- **Prior workup:** [what's been tested, results if known]
- **Prior specialist assessments:** [who's been seen, their conclusions]
- **Already ruled out:** [formal exclusions]
- **Known exam findings or unusual features:** [if any]
- **Functional impact:** [what they can/can't do]
- **Information gaps:** [list what's missing or unknown — be explicit]
```

**Critical:** In the "Information gaps" section, explicitly list fields that are unknown or thin. This serves two purposes — it tells the downstream analysis what it can't rely on, and it tells the user's physician what additional workup would sharpen the differential.

### Step 2.2 — Confirm with the user

Show the clinical summary to the user with this framing:

> "Here's what I understood from our conversation. Before I run the full research pipeline, I want to make sure I've got the important pieces right. Please read through this and tell me:
>
> - Anything I got wrong
> - Anything I missed that you think matters
> - Anything you'd phrase differently
>
> If it's good, just say 'go' or 'run it' and I'll proceed."

Wait for the user's response. Edit the summary based on their corrections. Repeat this confirmation step at most twice — more than that and you're spinning.

When the user confirms, proceed to Phase 3.

---

## PHASE 3 — Diagnostic analysis

This phase runs the same workflow as the benchmark's `opus-vanilla-with-tools` condition, which scored **76.0% Top-1 / 83.8% Top-5** on a 371-case post-cutoff rare-disease cohort run under contamination filtering (Wilson 95% CI for Top-1: [71.4%, 80.1%]; honest interpretation: mid-to-high 70s, given run-to-run variance — see the methodology footer). The analysis proceeds as a single grounded reasoning pass with heavy tool use — across the benchmark, the average run made ~23 tool calls per case.

You have access to:

- **HPO phenotype tools** (`search_hpo_terms`, `lookup_diseases_by_phenotypes`, `get_disease_phenotypes`, `phenotype_differential_diagnosis`) — for mapping the patient's clinical features to standardized phenotype codes, finding diseases whose canonical phenotype sets match, and confirming whether a candidate's expected features are present in this case
- **PubMed search tools** (`pubmed_search_articles`, `pubmed_fetch_articles`, `pubmed_lookup_mesh`, etc.) — for finding published case reports with similar multi-feature presentations, mechanism reviews, and clinical criteria papers
- **WebSearch** — for Orphanet and OMIM entries, gene-disease databases, and clinical criteria documents not indexed in PubMed

### Step 3.0 — Tool availability check

Before starting the analysis, verify that the expected MCP tools are actually available in this session. Run a quick probe of each tool family:

- HPO: call `search_hpo_terms` with a benign query (e.g., a common HPO term like "hypertension") — should return at least one hit.
- PubMed: call `pubmed_search_articles` with a benign query (e.g., "rare disease review") — should return at least one article.

If either probe fails (tool not found, or tool errors out), surface the gap to the user **before** producing a differential. Suggested message:

> Heads up: the `<HPO|PubMed>` MCP server isn't responding in this session. I'll proceed with WebSearch fallback, but the differential will be less reliable — case-report retrieval and citation handling will be hand-rolled instead of structured. If you want the full pipeline, the most common cause is the PubMed MCP server failing to register on a fresh install; the AGENT_INSTALL.md troubleshooting section covers the fix.

Then continue with the analysis. Don't silently fall back — the user should know which tools were actually available for their consult.

### Step 3.1 — Extract clinical features and map to HPO

Read the confirmed clinical summary. Identify every distinct clinical feature the patient has — symptoms, timeline patterns, exam findings, lab abnormalities, demographic context. For each feature, search HPO for the matching standardized term using `search_hpo_terms`. Record the HPO code and the canonical phenotype name.

Aim for breadth here. Rare disease diagnosis is usually about matching an unusual *constellation* of features, so capture features that seem secondary or "obvious" too — not just the chief complaint. If a feature is ambiguous (e.g., the patient said "weird bruising"), try multiple candidate HPO mappings (easy bruising vs. petechiae vs. ecchymoses) and keep the best fit.

If `search_hpo_terms` returns no good match for a described feature, note that explicitly — that feature won't drive the phenotype lookup but may still matter for pattern recognition.

### Step 3.2 — Look up candidate diseases by phenotype

Using the HPO codes from step 3.1, call `lookup_diseases_by_phenotypes` and `phenotype_differential_diagnosis` to find diseases whose canonical phenotype profiles overlap with the patient's. Pay attention to:

- **Match strength** — how many of the patient's features are core canonical features of each candidate (vs. variable or absent)
- **Coverage** — what fraction of the candidate's canonical features the patient actually has (a candidate that requires 6 cardinal features but the patient has only 1 is a poor fit)
- **Specificity** — features that are unusual or pathognomonic carry more weight than common nonspecific ones (fatigue, headache)

Build an initial working list of 10–20 candidates. Don't filter aggressively yet — keep candidates that are partial matches as well as strong ones; you'll prune in step 3.4.

### Step 3.3 — Query PubMed for related cases and mechanism

For your top candidates from step 3.2 — and for any unusual feature constellations not well-covered by HPO lookup — query PubMed for:

- **Case reports** with similar multi-feature presentations. The single strongest signal in rare disease diagnosis is often "has anyone reported this exact picture before?"
- **Clinical criteria papers** for any candidate where formal diagnostic criteria exist
- **Mechanism reviews** when reasoning about whether a candidate's pathophysiology could plausibly produce the observed cluster

For each PubMed hit you use as evidence, record the PMID. You'll cite these in the final output.

Use WebSearch to fill gaps PubMed can't — Orphanet's full phenotype list for a candidate, OMIM's mechanism description, or guideline documents from disease-specific foundations.

### Step 3.4 — Synthesize the top-5

With the HPO-driven candidate list, the PubMed evidence, and any WebSearch findings, build the final top-5 differential. For each candidate, evaluate:

- **Phenotype fit** — how many of the patient's HPO-mapped features are canonical for this disease, and how many of the disease's expected canonical features are present in the patient
- **Pattern fit** — does the case match a recognizable clinical gestalt? Is there a published case report with a closely matching constellation?
- **What doesn't fit** — features in the patient's picture that are atypical for this candidate, or canonical features of the candidate that are absent from the patient. Be honest here; an unfit feature is a real signal.
- **Differential exclusion** — for the top candidate, briefly consider what else this could be that would change the workup. Did anything get ruled out earlier in the user's odyssey that affects this?

Rank by overall fit. Aim for a mix: high-confidence convergent candidates where multiple evidence sources point the same way, plus 1–2 "worth considering" candidates that are partial matches but plausible enough to ask about.

If you cannot produce 5 candidates with concrete grounding (HPO codes or PMIDs), output fewer. Say "I found 3 candidates with solid supporting evidence" rather than padding.

---

## PHASE 4 — Presentation

Produce **one unified output** that serves both the patient and the doctor — written like a clinical-note hybrid: a tight summary the patient can scan in 30 seconds, then per-diagnosis detail with reasoning, supporting evidence, and links a physician can review. **Do NOT split into separate patient-facing and doctor-facing sections.** A single document that both audiences can use, with the summary up top.

### Step 4.1 — Caveats up front (verbatim)

```
## Before the differential — read this first

This is research, not a diagnosis. It's meant to help with **one step in a long medical journey** — giving you something concrete to discuss with a doctor. It is not the journey itself and not a substitute for one.

A few things worth holding in mind before you read the differential below:

- **It's often wrong.** The underlying method is right about its top candidate roughly 3 of 4 times on a held-out benchmark of rare-disease cases (mid-to-high 70s Top-1 with run-to-run variance); about 1 in 6 times, the correct answer isn't on the top-5 list at all. Read these as possibilities to ask about, not as the answer.
- **You might not have a rare disease at all.** This tool is purpose-built for *rare* differential diagnosis — it will produce a confident-looking rare-disease list even when your symptoms are actually explained by something common (depression, thyroid, anemia, sleep apnea, medication side effects, deconditioning, sleep deprivation). Your doctor will rightly consider those first.
- **Bring this as questions, not conclusions.** "I've been wondering about Disease X — does that fit my picture?" is more useful than "I think I have X." Let your doctor run their own differential; this list is a research aid, not a verdict.
- **If you feel anxious reading this** — especially the more serious candidates — consider going through it with a trusted person rather than alone late at night. Rare-disease differentials often include progressive conditions that are hard to read about without medical context.
- **Verify any reference you'd act on.** The model is grounded in real HPO codes and PubMed citations, but can occasionally misrepresent what a paper actually says.

What your doctor brings that this can't: a physical exam, your full chart, pre-test-probability judgment for *your* situation, and pattern recognition across many patients. The output below is most useful as raw material for them.
```

### Step 4.2 — Top-5 summary at the top

A scannable table that lets a reader (patient or doctor) see the whole picture in 10 seconds. Each row has the disease name, a confidence tag, and a one-line "why this is here." Confidence tags use this scale — pick exactly one per candidate based on how well the evidence converges:

| Tag | Use when |
|---|---|
| **Strong match** | Multiple core canonical features present (HPO-confirmed); a published case report or clinical criteria document closely matches the picture; few or no non-matching features |
| **Plausible** | Most of the canonical picture fits with HPO support, but one or two notable gaps remain, OR phenotype overlap is strong but no closely-matching case report was found |
| **Worth considering** | Partial phenotype match — some core features present but significant ones missing or not yet evaluated; included because the partial fit is specific enough to be worth asking about |
| **Long-shot** | Weak overall fit but the candidate is included because a single highly specific feature (or an unusual feature constellation) points at it and the consequences of missing it would be significant |

Output the summary block exactly like this (no extra prose between caveats and summary):

```
## Top 5 to discuss with your doctor

| # | Diagnosis | Confidence | One-line summary |
|---|-----------|-----------|------------------|
| 1 | **[Disease name in plain English]** *([technical name if different])* | [Strong match / Plausible / Worth considering / Long-shot] | [Single sentence: what makes this fit] |
| 2 | ... | ... | ... |
| 3 | ... | ... | ... |
| 4 | ... | ... | ... |
| 5 | ... | ... | ... |
```

### Step 4.3 — Per-diagnosis detail (one section per candidate)

After the summary table, give a clinical-note-style section per candidate. Write it so a patient can follow the reasoning AND a doctor can verify the evidence in under 60 seconds per candidate. Use this template for every top-5 entry — don't skip sections, but keep each section tight (3-5 lines max where feasible):

```
### [#]. [Disease name in plain English] — [Confidence tag]

*Technical name:* [Canonical clinical name if different] · *Orphanet:* [ORPHA:xxxxx or "not coded"] · *OMIM:* [OMIM:xxxxxx if applicable]

**What it is** — *(1-2 sentences, plain language, then 1 sentence on mechanism for the clinician)*
[Plain-language explanation. Then: "Mechanistically, [the underlying pathophysiology in one sentence — what's broken at the molecular or organ-system level]."]

**Why it's on the list (matching features)**
- [Patient's specific symptom / finding] → matches canonical feature *([HPO code if available])*. [Source: PMID:xxx | URL]
- [Patient's symptom / finding] → matches canonical feature *([HPO])*. [Source: PMID:xxx | URL]
- [Timeline or pattern detail] → matches canonical feature *([HPO])*. [Source: PMID:xxx | URL]

**What doesn't fit (be honest)**
- [Feature in the patient's picture that's atypical for this diagnosis] — [why this matters; how unusual it is]
- [Canonical feature of the diagnosis that's absent or unknown in this case] — [whether absence rules it out or just weakens the fit]
- [If everything fits, say so explicitly: "No major non-matching features identified."]

**Evidence summary**
- *HPO features matched:* [N of the patient's mapped features hit canonical phenotypes for this disease, e.g. "4 of 6 patient features"]
- *Canonical-feature coverage:* [fraction of the disease's expected core features present in the patient, e.g. "5 of 8 core features present, 2 unknown, 1 absent"]
- *Closest published case report:* [PMID + 1-line description, or "no closely-matching case report found"]

**To confirm or rule out**
- *Tests to discuss:* [specific test name, e.g., "Anti-titin antibody panel"]. [Why this test: 1 sentence on what positive/negative result would mean.]
- *Specialist to consult:* [e.g., "Rheumatology" or "Cardiology — ideally one with experience in [X]"]. [Why: 1 sentence.]
- *What to track in the meantime:* [Symptom diary item or pattern to watch for; only if there's something specific].

**Sources**
- [PMID:xxxxxxx — Title (Year)](https://pubmed.ncbi.nlm.nih.gov/xxxxxxx/)
- [PMID:xxxxxxx — Title (Year)](https://pubmed.ncbi.nlm.nih.gov/xxxxxxx/)
- [Orphanet entry](https://www.orpha.net/en/disease/detail/xxxxx) | [OMIM entry](https://omim.org/entry/xxxxxx)
- [Additional source if cited: URL — Title]
```

Repeat the per-diagnosis section for all 5 candidates, ordered by rank. Don't drop sections — if a piece of info isn't available (e.g., no Orphanet code, no OMIM), say so explicitly rather than omitting the line.

### Step 4.4 — Methodology + next-steps footer

After the 5 detail sections, append the methodology note + next-steps. Patient-and-doctor neutral — useful as audit trail and as patient guidance:

```
---

## How this was generated

This differential was produced by a single Claude agent working the case end-to-end with three tool families: the Human Phenotype Ontology (for mapping symptoms to standardized phenotype codes and looking up diseases by phenotype), PubMed (35M+ biomedical articles, for case reports and clinical criteria), and general web search (for Orphanet, OMIM, and other gene-disease references). The agent extracted clinical features from the intake, mapped each to HPO, used phenotype-based lookup to surface candidate diseases, queried PubMed for similar published presentations, and synthesized a grounded top-5 — typically making ~20-25 tool calls in the process.

The same workflow was validated at **76.0% Top-1 / 83.8% Top-5** on a 371-case post-cutoff rare-disease benchmark cohort run under contamination filtering — source-paper retrieval at inference time blocked (Wilson 95% CI for Top-1: [71.4%, 80.1%]). For comparison, Claude answering the same cases with no tools and no extended thinking scored 47.7%; Claude with extended thinking but no tools scored 54.9%. The +21.1pp lift from tools over extended-thinking-only is statistically significant (McNemar p<0.0001). This is a held-out cohort where the cases are dated after the model's training cutoff, so familiarity with the specific published cases is minimized — though general training-time exposure to the disease entities themselves still applies. Caveat: single-shot agent benchmarks at this scale carry substantial run-to-run variance (a 25-case rerun showed ~28% per-case verdict flip rate); the headline number is best read as "mid-to-high 70s on this cohort," not as a precise 76.0%.

## What to do with this

1. **Bring this whole document to your next appointment.** The summary table is what your doctor will scan first; the per-diagnosis detail is there if they want to dig in.
2. **Ask your doctor to comment on the top 3.** A quick "yes that's plausible, let's test for it" or "no, here's why" is the valuable signal. Don't argue which one is right — that's their call with your full picture.
3. **If none of these feel close, tell your doctor that too.** The research covered one possibility space; there are others, and a doctor's broader differential matters.
4. **Track any new symptoms** between now and your appointment. Rare disease diagnosis often comes from a new feature appearing that narrows the field.
5. **Remember the base rate.** Common things are common. This research specifically looked for *rare* possibilities — your doctor will (correctly) consider common explanations first. These possibilities are only worth pursuing if common ones have been ruled out or don't fit.

## Information gaps that would sharpen this

If the user came in with incomplete information (no family history, missing labs, no formal rule-outs), surface a short bulleted list here of "what additional information would tighten this differential." These are things the patient could surface at the next visit OR bring back to a follow-up consult with this tool.

- [Specific gap, e.g., "Family history of immune disorders — not yet asked"]
- [Specific gap, e.g., "Anti-nuclear antibody panel result — patient mentioned bloodwork but not specific values"]
```

### Step 4.5 — Source materials ledger

After the methodology and next-steps, append a deduplicated index of everything cited across the document, so a doctor can verify any single source in one place. The ledger collects every HPO code, PMID, and URL the agent cited as evidence during Phase 3:

```
---

## Source materials ledger (deduplicated)

**HPO codes referenced**
- HP:xxxxxxx — [phenotype name]
- HP:xxxxxxx — [phenotype name]
- ...

**PubMed references**
- [PMID:xxxxxxx — Title (Authors, Year)](https://pubmed.ncbi.nlm.nih.gov/xxxxxxx/)
- ...

**Orphanet / OMIM entries**
- [ORPHA:xxxxx — Disease name](https://www.orpha.net/en/disease/detail/xxxxx)
- [OMIM:xxxxxx — Disease name](https://omim.org/entry/xxxxxx)
- ...

**Other web sources**
- [Title — domain](URL)
- ...
```

### Step 4.6 — Final disclaimer

Always append this to the bottom of the output, verbatim:

```
---

**Disclaimer.** This output was generated by Claude using a structured research workflow and live queries against the Human Phenotype Ontology, PubMed, and general web search. It is research assistance, not medical advice, diagnosis, or treatment recommendation. Accuracy is bounded by the completeness of the input and the limits of the underlying knowledge bases; rare disease diagnosis in real patients requires physician evaluation, physical examination, and often specialized testing. Do not act on this output without physician review. If anything described suggests a medical emergency, contact emergency services immediately.

The methodology underlying this skill is open-source at github.com/fryanpan/rare_disease_benchmark and was validated at 76.0% Top-1 / 83.8% Top-5 on a 371-case post-cutoff rare-disease benchmark cohort run under contamination filtering (Wilson 95% CI for Top-1: [71.4%, 80.1%]; honest interpretation: mid-to-high 70s, given substantial run-to-run variance on agent runs).
```

---

## Groundedness requirements — non-negotiable

Every disease in the final top-5 MUST have at least one of:
- A specific HPO code matching a specific feature from the user's intake
- A specific PubMed reference (PMID) found during Phase 3 tool calls
- A specific WebSearch source (URL) found during Phase 3 tool calls

If a candidate has no concrete source citation, **drop it from the top-5** and replace with the next-best candidate that does. Speculation without source is not useful to either the patient or the physician.

If after this filtering you have fewer than 5 candidates with solid citations, output fewer — say "I found 3 candidates with solid supporting evidence" rather than padding.

---

## What to do if the analysis fails

- **If HPO lookup returns empty (no clear phenotype mapping for the user's symptoms):** say so. "The HPO lookup didn't find clear matches for the symptoms described, which usually means either the symptoms are described too informally or the combination is unusual. The differential below is based more on pattern recognition and PubMed case-report searching than phenotype matching." This is important context for the user.
- **If PubMed returns no closely matching cases for any candidate:** still produce the differential from HPO + general knowledge, but mark each candidate's "Closest published case report" line as "no closely-matching case report found" rather than omitting it. The absence is itself information.
- **If a tool errors or times out repeatedly:** say so explicitly. "PubMed search was unavailable during this consult; the differential below is grounded in HPO matches only." Don't pretend a missing data source was used.
- **If the user's presentation is too thin for meaningful analysis:** say so. Don't generate a differential of well-known conditions with no distinguishing support — that's worse than saying "I can't give you a useful answer."

---

## Length, tone, format

- Plain-language sections: warm, honest, no medical jargon without immediate definition, no false reassurance, no dramatization
- Clinical sections: terse, professional, citation-heavy
- Never produce more than 5 candidates in the final differential — more than 5 is noise, not information
- Never withhold a candidate because it sounds scary if it fits the evidence — but DO frame it with appropriate uncertainty
- Never recommend a treatment or tell the user to change medications — only tests and specialty referrals
