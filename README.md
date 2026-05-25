# rare-disease-consult

A Claude Code plugin that runs a structured rare-disease diagnostic consultation. Claude does a guided interview about your symptoms, looks them up in two free medical databases (HPO and PubMed), and produces a ranked top-5 list of possible diagnoses with citations you can verify.

## What it does

This plugin runs a single Claude session end-to-end on a rare-disease case, with access to two open public medical databases:

- **HPO (Human Phenotype Ontology)** — a standardized vocabulary of medical signs and symptoms
- **PubMed** — the NIH medical-literature database (~35 million articles)

Claude does a structured interview to gather your symptoms, history, and what's already been tried or ruled out, looks up which rare diseases match your symptoms, searches PubMed for similar published case reports, and produces a top-5 list with concrete citations.

## Quick install

In a Claude Code session, paste this and press enter:

> Please install the `rare-disease-consult` plugin from `https://github.com/fryanpan/rare-disease-consult-plugin`. Follow the instructions in its [`AGENT_INSTALL.md`](https://github.com/fryanpan/rare-disease-consult-plugin/blob/main/AGENT_INSTALL.md) file.

Claude handles the safety disclaimer, prerequisite checks, clone, and registration. After restart, run `/rare-disease-consult:consult`.

**Currently Claude Code only** — Cowork support is planned. For the manual install or first-run troubleshooting, see [Installation](#installation) below.

## How it works

There are four phases:

1. **Intake.** Claude has an open-ended conversation to gather your symptoms, history, family history, and what's already been tried or ruled out.
2. **Summary.** Claude composes a structured clinical summary and shows it to you to confirm before going further.
3. **Analysis.** Claude looks up your symptoms in HPO, queries PubMed for similar published case reports, and synthesizes a top-5 list of candidate rare diseases. Each candidate is required to have concrete citations (HPO codes, PubMed IDs, or URLs); candidates without grounding are dropped. An average run makes ~23 tool calls per case.
4. **Output.** A single document with three pieces:
   - **Top-5 summary table** at the top — 10-second skim for you or a doctor.
   - **Per-diagnosis sections** with what fits your picture, what doesn't, citations, and tests or specialists to discuss.
   - **Consolidated source list** at the bottom so any single citation can be verified in one place.

## Benchmark results

On a benchmark of 371 cases the model couldn't have memorized — published after Claude's training data was frozen, with safeguards against looking up the answer at runtime — this workflow scored about **76% Top-1 / 84% Top-5** (vs ~26% for a general physician at first visit). The improvement is statistically significant vs Claude without tools (47.7%) and Claude with thinking but no tools (54.9%).

One important caveat: agent runs vary run-to-run. A 25-case re-run under identical conditions agreed with itself on only 72% of cases. **Read the tools number as "mid-to-high 70s," not as a precise 76%.**

See [Benchmark provenance](#benchmark-provenance) below for the full methodology.

## How to use this responsibly

**This plugin helps with one step in a long medical journey — not the whole journey.** If your symptoms have been hard to pin down, a structured differential from this tool can give you something concrete to walk into a doctor's office with. That's the use case. It isn't more than that.

### What this tool is FOR

- **A starting point for an informed conversation with your doctor.** Walking in with "I've been wondering about Disease X — does that fit my picture?" is more productive than "I have these symptoms, what is it?"
- **Help finding the right *type* of specialist to see.** If the differential keeps surfacing immunology candidates, that's a signal to consider an immunologist over a general internist.
- **A second-opinion-style sanity check** when you've been told "we don't know" or "it's probably nothing" and your gut says otherwise.

### What it is NOT

- **Not medical advice.** Only a physician can diagnose. Only a physician can prescribe, order tests, or rule things out responsibly.
- **Not a substitute for urgent care.** If you have red-flag symptoms (chest pain, stroke signs, sudden severe headache, sepsis features, etc.), go to the ER. The skill has a safety check that redirects, but don't rely on it as a triage layer.
- **Not a treatment recommender.** The skill never recommends starting, stopping, or changing medications. If it points at a condition, the next step is a doctor — not a supplement or off-label medication.

### Risks worth taking seriously

Even when the tool surfaces a plausible-looking differential, several failure modes can hurt you:

- **It's often wrong.** Top-1 was in the mid-to-high 70s on the benchmark, which means roughly 1 in 4 times the first candidate isn't the right answer. Top-5 was about 84% — still about 1 in 6 cases where the right diagnosis isn't on the list at all.
- **You might not even have a rare disease.** This tool is purpose-built for *rare* differential diagnosis. It will produce a confident-looking list of rare diseases for input that's actually explained by something common (depression, thyroid, anemia, sleep apnea, medication side effects, deconditioning, etc.). Rare conditions are rare for a reason — chasing one when the cause is common wastes time, money, and emotional energy.
- **Anchoring bias on your doctor.** Walking in with a confident AI-generated list can prime the doctor to evaluate those specific candidates rather than running their own broader differential. Present it as "things I've been wondering about" — not "I think I have X."
- **Workup cascade harm.** Pursuing the wrong rare disease can mean expensive specialist visits, invasive testing (biopsies, genetic panels), false positives from sensitive tests, and months of stress and uncertainty. Make sure each follow-up test is one your doctor agrees is worth doing for your specific situation.
- **Cyberchondria and anxiety.** Rare-disease differentials often include serious progressive conditions. Reading the scariest candidates without medical context can cause real distress. If you're already anxious about your symptoms, consider going through the output with a trusted person rather than alone late at night.
- **Hallucinated citations.** Despite the HPO and PubMed grounding, the model can occasionally misrepresent what a paper actually says. Verify any citation you plan to act on by reading the underlying abstract on PubMed.
- **Privacy.** Whatever you type goes to Anthropic. Per their data policy, prompts and outputs may be retained. If you have specific concerns about insurance, employment, or other downstream uses, factor that into what you share.

### What a doctor brings that this tool cannot

A good clinician contributes a lot more than a ranked list:

- **Understanding your specific needs and life context.** What you can tolerate, what tradeoffs you'd accept, what tests you can or can't afford, what living with treatment X actually means for your life.
- **A physical exam and the things only an in-person assessment reveals** — subtle findings, the gestalt of how sick you look, the parts of your story you didn't think to share.
- **Tailoring the confirmatory diagnostic process** to your situation — choosing the right test sequence, weighing pre-test probabilities, deciding when to watch-and-wait versus when to escalate.
- **Tailoring treatment** if a diagnosis lands — which option fits your life, your other conditions, your goals.
- **Pattern recognition across hundreds of similar patients** — something an LLM approximates but doesn't truly have.

The output of this plugin is most useful as raw material for a clinician who knows you. The clinician is the synthesizer; this is a research aid.

## Installation

The fastest path is the [Quick install](#quick-install) above — paste one prompt into Claude Code and the agent handles the rest. The manual path below is for users who want to clone and register themselves, or for troubleshooting.

Prerequisites:

- [Claude Code](https://code.claude.com) installed and authenticated
- [`uv`](https://docs.astral.sh/uv/) — single-binary Python package manager. Install: `curl -LsSf https://astral.sh/uv/install.sh | sh` (macOS/Linux) or [docs.astral.sh/uv](https://docs.astral.sh/uv/) for Windows.
- `node` / `npx` — [nodejs.org](https://nodejs.org/) or your platform's package manager.

Then:

```bash
git clone https://github.com/fryanpan/rare-disease-consult-plugin.git rare-disease-consult
cd rare-disease-consult
claude plugin install .
```

Restart Claude Code, then run `/rare-disease-consult:consult`.

**First-run note:** the HPO lookup tool downloads ~50 MB of HPO + Orphanet annotation data on first invocation (cached after that, ~30-second one-time delay).

## Usage

### Direct invocation

```
/rare-disease-consult:consult
```

Claude will open the conversation with an open-ended intake question. Describe what's been happening — timeline, symptoms, family history, what's been tried, what's been ruled out. Claude will follow up on gaps, synthesize a structured summary for you to review, and then run the analysis with live HPO and PubMed lookup.

### Auto-invocation

The skill also auto-invokes when Claude detects a user describing undiagnosed symptoms, a long search for a diagnosis, or asking "what could this be?" about a hard-to-explain symptom cluster. You don't need to type `/rare-disease-consult:consult` — just start the conversation.

## Example session

```
You: I've been trying to figure out what's wrong with me for 3 years. I'm
     39, female. Started with episodes of severe joint pain that came and
     went. Then fatigue. Then my skin started bruising easily. My doctor
     tested for lupus — negative. Rheumatologist thought maybe fibromyalgia
     but said my joints don't feel right for that. My mom had "something
     weird with her connective tissue" that was never really diagnosed.
     What could this be?

Claude [invokes rare-disease-consult:consult]:
     [Opens intake, asks follow-up questions about specific joint patterns,
      skin findings, any hypermobility, family history of dissections
      or aneurysms, height, etc.]

[... intake continues ...]
[... clinical summary presented and confirmed ...]
[... Claude extracts features, queries HPO, looks up diseases by phenotype,
     searches PubMed for related case reports ...]
[... synthesizes a grounded top-5 differential ...]

Output: Top-5 summary table, per-diagnosis detail with HPO codes and PMIDs,
        and a deduplicated source-materials ledger.
```

## How this plugin works under the hood

*(Engineer section.)* The plugin has three components:

1. **Main skill** (`skills/consult/SKILL.md`) — a long-form instruction file that tells Claude how to run the four-phase workflow. The frontmatter `description` triggers auto-invocation. This is where the diagnostic-analysis prompt logic, safety checks, and presentation format live.
2. **HPO MCP server** (`server/hpo_server.py`) — a custom FastMCP server that wraps [pyhpo](https://github.com/Centogene/pyhpo) for HPO+Orphanet disease lookups and the NLM Clinical Tables API for free-text symptom-to-HPO-term mapping. Four tools: `search_hpo_terms`, `lookup_diseases_by_phenotypes`, `get_disease_phenotypes`, `phenotype_differential_diagnosis`.
3. **PubMed MCP** — uses [@cyanheads/pubmed-mcp-server](https://www.npmjs.com/package/@cyanheads/pubmed-mcp-server) via `npx` — nine tools for searching PubMed's 35M+ biomedical articles, free and no auth required.

## Benchmark provenance

This plugin packages the exact setup that scored **76.0% Top-1 / 83.8% Top-5** on a 371-case test set published after Claude's training cutoff, with an **answer-blocking filter on** (Claude can't retrieve the actual source paper for each case at runtime). The honest interpretation is "mid-to-high 70s" — see the run-to-run variance caveat above. Same setup *without* the answer-blocking filter scored 65.5% on the same cases; the 10-point gap is mostly run-to-run noise, not a clean filter effect.

See [METHODOLOGY.md](../../METHODOLOGY.md) in the parent repo for:

- How the test set was built and how diagnosis-framing language was stripped
- How the workflow compares to Claude without tools and Claude with thinking only
- How the answer-blocking filter works and why we report the filtered number
- How sample sizes and statistical significance were chosen
- How the test set compares to the broader benchmark literature

This plugin is **MIT licensed**. It contains no RareArena-derived material. The parent benchmark repo's evaluation code is under CC BY-NC-SA 4.0 (RareArena's license); that constraint does not flow through to this plugin.

## What we tried that didn't work

We also tested a multi-agent setup — three specialist sub-agents reasoning independently, then voting after seeing each others' anonymized answers. It looked promising on the original benchmark, but with the answer-blocking filter on, the multi-agent score dropped 12 points (single-agent's didn't drop). It also costs 5–10× more per case — each sub-agent makes its own tool calls — for no measurable improvement. The single-agent workflow is what shipped.

## Known limitations

- **Memorization isn't fully controlled.** The model may have seen some published rare-disease cases during training. We report on a test set of cases published after Claude's training data was frozen to minimize this, but general training-time exposure to the *diseases themselves* still applies. Real-patient performance — where the case isn't in any public corpus — would likely be lower than the benchmark score.
- **Claude doesn't agree with itself run-to-run.** A 25-case re-run agreed with itself only 72% of the time. Tool-using agents vary because they make different choices about which tools to call each run. Treat 76% as "mid-to-high 70s," not a precise number.
- **The HPO server has an IPv6 workaround.** It forces IPv4 because NLM's IPv6 endpoint is unreachable from some networks. If you hit timeouts on symptom-to-HPO lookups, check that this patch is active.
- **Each consultation costs a couple of dollars.** A run averages ~23 tool calls with a long reasoning context — expect $1–3 of Anthropic API spend per use on Opus.
- **Not tested on pediatric or psychiatric cases.** The benchmark is mostly adult internal-medicine cases. Other populations are untested.

## License

**MIT License.** See [LICENSE](LICENSE).

This plugin is independently developed and freely usable for both commercial and non-commercial purposes with attribution. The plugin contains no RareArena-derived code; the citations of RareArena throughout the docs are benchmark references, not derivations.

(The parent `rare_disease_benchmark` repo's eval harness — `eval_condition.py`, `metrics.py` — is separately licensed under CC BY-NC-SA 4.0 because its evaluation prompts are adapted from RareArena's `eval_updated.py`. That constraint applies only to the benchmark code, not to this plugin.)

## Credits

- **HPO / Orphanet data:** via [pyhpo](https://github.com/Centogene/pyhpo) (Centogene) and the [NLM Clinical Tables API](https://clinicaltables.nlm.nih.gov/).
- **PubMed MCP:** [@cyanheads/pubmed-mcp-server](https://github.com/cyanheads/pubmed-mcp-server).
- **Benchmark dataset:** [RareArena](https://github.com/zhao-zy15/RareArena) (Lancet Digital Health 2026).
