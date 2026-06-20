# Ai201 Project 3 — TakeMeter: Classifying Football 

A 3-way text classifier categorizes posts from the **r/football** public into three categories: **`analysis`**, **`hot_take`**, and **`reaction`**. The aim is to provide a tool that can automatically label or sort posts based on intent. The results will allow users to quickly and easily differentiate between a thoughtful, fact-based argument and an opinionated, unfounded statement, and an emotion-driven response to an event as it unfolds.

The complete upstream design - including community selection, label definitions, edge case definitions, data collection plan, metrics, and success criteria - can be found in [planning.md](planning.md). This README serves as both the **implementation documentation** and the **evaluation report.**

---

## 1. Community & Task

**Community:** r/football (~16M subscribers), a general association-football subreddit.

**Why it fits a classification task:** The feed includes three different *intents* within the messages: logical statement, assertive opinion and visceral experience; however, members of the community have already somewhat organized the messages using counter-arguments like *"where's your proof?"*, *"that’s another controversial take."*, and *"what's your evidence?"*. For a community tool, the most important difference between reactions and hot takes is that both types are light on evidence, but they are perceived to have different levels of evidentiary requirement and the same model affecting both will incorrectly depict to the community how it communicates. (See [planning.md S1](planning.md.))

---

## 2. Labels (refined from the labeled data)

These are the definitions that actually separated the 206 labeled posts; they are the improved definitions of the original
Section-2 definitions and reflect the changes made during label stress-testing and annotation
(see [planning.md S8](planning.md)).

| Label | One-line definition | Decision test |
|---|---|---|
| **`analysis`** | Conclusion **carried by verifiable, load-bearing evidence** — stats/metrics, historical records, *or* structured named reasoning. Remove the evidence and the claim collapses. | Is there load-bearing, verifiable support? Naming players alone is **not** evidence. |
| **`hot_take`** | A **bold, confident, portable claim** asserted **without verifiable evidence** — the claim is its own justification. Covers people **and** the game itself (rules, format, officiating). | Strip the event/emotion: does a claim you'd still defend a week later remain? → **hot_take** |
| **`reaction`** | A post driven by **expiring, event-bound emotion** — it exists only because something just happened. Strip the event and the emotion and **nothing defensible remains**. | Strip the event/emotion: if only an expiring feeling remains → **reaction** |

**The hard boundary — `hot_take` ↔ `reaction`:** *portable claim vs. expiring emotion.* This was
pre-registered in planning as the distinction the model would most likely struggle with, and the
evaluation below tests exactly that.

### Examples (2 per label)

**`analysis`**
1. *"Under Slot, Liverpool's PPDA sits at 4.8 — 3rd in the PL and higher intensity than Klopp's last full season (6.2). The press isn't weaker, it's actually more aggressive, just more zonal. Media narrative is wrong."* — conclusion rests on verifiable metrics; remove the PPDA figures and the claim collapses.
2. *"Lewandowski's UCL knockout conversion rate (24%) vs Ronaldo's in the same rounds (19%) over the last decade — people say Lewandowski choked in Europe but the finishing numbers don't back that up."* — historical record is load-bearing; the stats *are* the argument.

**`hot_take`**
1. *"Mbappé has never been a big-game player and never will be. Champions League exposes him every time."* — bold, portable claim about a player with no verifiable support; the assertion is its own justification.
2. *"Hydration breaks are ruining the flow of the modern game — they should just be scrapped."* — a confident claim about *the game itself* (rules/format), defensible a week later, no event required.

**`reaction`**
1. *"THAT MBAPPE GOAL. The angle from that distance? How is that even physically possible??"* — expiring, event-bound emotion; strip the goal and nothing defensible remains.
2. *"Up 2-0 with 15 minutes left and we still lost 3-2. Same script every season. I'm done."* — exists only because of the result just witnessed; pure in-the-moment venting.

---

## 3. Dataset

- **Source:** r/football text posts (manual export), collected broadly rather than cherry-picked.
- **Size:** **206 labeled posts** in [football_posts_data.csv](football_posts_data.csv), columns
  `post_id, post_text, label, edge_notes`.
- **`edge_notes`** (48 posts annotated) records which decision rule fired on each borderline case, so
  annotation consistency can be audited later.

**Labeling process.** Every post was assigned a label by me as the human annotator using the refined
§2 decision tests. Most posts were unambiguous and labeled directly. For the borderline cases I could
not cleanly separate, I used an LLM (Claude) to produce a *pre-label* based on the ambiguous features
of that specific post (planning S7b); I then reviewed each pre-label and made the final call myself,
recording the decision rule that fired in `edge_notes`. The pre-labels were a tie-breaking aid only —
**every final label was assigned by human review**, not the model. This kept the `hot_take ↔ reaction`
boundary (the pre-registered hard case) annotated consistently against a single written rule. See the
[AI Usage](#11-ai-usage) section for the annotation-assistance disclosure.

| Label | Count | Share | Mean length (chars) | Median length |
|---|---|---|---|---|
| `hot_take` | 82 | 40% | 125 | 114 |
| `reaction` | 69 | 33% | 125 | 113 |
| `analysis` | 55 | 27% | **219** | **188** |

For evaluation, there are two important pieces from this table: (1) hot_take has the plurality of examples (40%), and (2) They are, on average, ~75% longer than either of the other classes. They both end up being a reason for why the model made the errors it did (section 6).

I decided to keep the natural class imbalance instead of fabricating examples of the rarer analysis class — as noted as a documented fallback plan in the Planning Doc. There is a training set of ~175 examples and a held-out test set of 31 examples.

---

## 4. Method

- **Fine-tuned model:** `distilbert-base-uncased`, fine-tuned as a 3-class sequence classifier
  (`{analysis: 0, hot_take: 1, reaction: 2}`).
- **Baseline:** a **zero-/few-shot LLM prompt** — Firstly, an LLM is given the Section-2 label definitions and
  asked to classify each test post, without fine-tuning. This is the baseline the fine-tuned model has to
  clear: does training a small dedicated model on our own annotations beat just prompting a general
  model with the rubric?
- **Evaluation:** both models scored on the **same 31-post held-out test set**. Artifacts:
  [evaluation_results.json](evaluation_results.json) and [confusion_matrix.png](confusion_matrix.png).

---

## 5. Evaluation Report

### 5.1 Overall accuracy

| Model | Accuracy | Test set |
|---|---|---|
| Baseline (zero-/few-shot LLM prompt) | **0.7742** (24 / 31) | 31 |
| Fine-tuned DistilBERT | **0.8387** (26 / 31) | 31 |
| **Improvement** | **+0.0645** | — |

Fine-tuning beat the prompt baseline by ~6.5 points of accuracy on the same test set. Both clear the
majority-class baseline (~40%) and the random-chance floor (~33%) comfortably.

### 5.2 Per-class metrics — Fine-tuned DistilBERT

Computed directly from the confusion matrix below.

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 1.000 | 0.625 | 0.769 | 8 |
| `hot_take` | 0.722 | 1.000 | 0.839 | 13 |
| `reaction` | 1.000 | 0.800 | 0.889 | 10 |
| **Macro avg** | **0.907** | **0.808** | **0.832** | 31 |
| **Accuracy** | | | **0.839** | 31 |

Asymmetry defines the tale: "analysis" & "reaction" exhibits **ideal accuracy but imperfect recollection**. "hot_take" offers **imperfect accuracy (0.722)**, yet has highest recollection; therefore, this indicates that analysis or reaction will **never** be misclassified, but will also **inadvertently** be equally misclassified if the model is unable to determine the distinction between them by being placed into the **hot_take** classification where the model is uncertainty.

### 5.3 Per-class metrics — Baseline (zero-/few-shot LLM prompt)

Evaluated on all 31 parseable responses.

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 0.71 | 0.62 | 0.67 | 8 |
| `hot_take` | 0.75 | 0.69 | 0.72 | 13 |
| `reaction` | 0.83 | 1.00 | 0.91 | 10 |
| **Macro avg** | **0.77** | **0.77** | **0.77** | 31 |
| **Weighted avg** | 0.77 | 0.77 | 0.77 | 31 |
| **Accuracy** | | | **0.774** | 31 |

### 5.4 Baseline vs. fine-tuned — what fine-tuning actually changed

| Class | Metric | Baseline | Fine-tuned | Δ |
|---|---|:---:|:---:|:---:|
| `analysis` | Precision | 0.71 | **1.00** | **+0.29** |
| `analysis` | **Recall** | **0.62** | **0.62** | **±0.00** |
| `hot_take` | Precision | 0.75 | 0.72 | −0.03 |
| `hot_take` | Recall | 0.69 | **1.00** | **+0.31** |
| `reaction` | Precision | 0.83 | **1.00** | +0.17 |
| `reaction` | Recall | **1.00** | 0.80 | **−0.20** |
| — | Macro-F1 | 0.77 | **0.83** | +0.06 |

Two findings stand out:

1. **Fine-tuning has improved precision at the expense of recall for `analysis`.**  The precision of a fine-tuned model that produces a `false alarm` to produce an `analysis` is 0.71 → 00; while the recall of a fine-tuned model that produces an `analysis` is the same as the baseline's recall (0.62). Neither model is able to catch approximately three analytical posts that are relatively difficult to analyze; therefore neither model demonstrates evidence via qualitative evidence of the thin/qual of evidence. Based upon the fine-tuned model's stricter definition of the original model's definition, it is clear that the failure is due to not having enough examples within the training set.
2. **The baseline and fine-tuned models produce distinctly different errors.**  A baseline model (LLM) that uses the rubric produces a diverse distribution of errors over the entire rubric.  This results in a possibility of a significant recall failure in the hot take analysis area with a recall rate of 0.69.  Because of the way the fine-tuning works, it deletes/combines errors made by the fine-tuned model into one (when combined). The fine-tuned model demonstrated a .06 net macro-F1 difference versus the baseline notification, but the overall distribution of errors produced by the either models in two distinct failure classifications did show greater failure rates in each failure classification.

### 5.5 Confusion matrix — Fine-tuned DistilBERT

Rows = true label, columns = predicted label (also in [confusion_matrix.png](confusion_matrix.png)).

|  | → pred `analysis` | → pred `hot_take` | → pred `reaction` | **Total (true)** |
|---|:---:|:---:|:---:|:---:|
| **true `analysis`** | **5** | 3 | 0 | 8 |
| **true `hot_take`** | 0 | **13** | 0 | 13 |
| **true `reaction`** | 0 | 2 | **8** | 10 |
| **Total (pred)** | 5 | 18 | 8 | 31 |

**Every off-diagonal cell sits in the `hot_take` column.** All 5 errors are misclassifications *into*
`hot_take` (3 `analysis → hot_take`, 2 `reaction → hot_take`), with zero errors in any other
direction. This confirms the pre-registered hypothesis: the model hasn't learned where `hot_take` ends.

---

## 6. Failure Analysis

Following planning S7c: I exported every wrong prediction, grouped them into candidate patterns, and
**verified each against the confusion matrix and actual post texts**, discarding any that didn't hold.
With only 5 errors, this is the *full* error set — not a sample, but every mistake the model made.

### 6.1 Patterns I tested, kept, and discarded

| Candidate pattern | Verdict | Evidence |
|---|---|---|
| **All errors collapse into one class (`hot_take`)** | ✅ **Kept** | Confusion matrix: all 5 off-diagonal cells are in the `hot_take` column. `hot_take` precision 0.722 vs. recall 1.000. |
| **One dominant directional pair (`analysis → hot_take`)** | ✅ **Kept** | 3 of 5 errors; the single most common error transition. |
| **Low-confidence / majority-class fallback** | ✅ **Kept** | Every error's confidence is **0.35–0.41** — barely above the 0.33 three-class floor. `hot_take` is the plurality training class (40%), so "unsure → default to `hot_take`" fits. |
| **Short / low-information `analysis` posts** | ✅ **Kept** | Misclassified `analysis` posts 120 (180 ch) and 48 (195 ch) are at/below the `analysis` median length (188); the model learned "long + numbers = analysis." |
| **Sarcasm drives the errors** | ❌ **Discarded** | Re-reading all 5: only post 178 uses hyperbole ("blood pressure through the roof"); none are sarcastic. Sarcasm is not the driver — assertive *framing* and missing stat-blocks are. |
| **A correction I had to make** | — | I initially matched the "Haaland" error to the wrong dataset row (post 14, a different Haaland post). Re-grepping on a longer snippet identified the real one (**post 36**). Pattern claims were re-checked against the corrected text. |

### 6.2 Three specific errors, analyzed

**Error 1 — `analysis → hot_take` (post 36, confidence 0.36)**
> *"Haaland's club numbers are insane, nearly a goal a game in the Prem — but Norway never going deep
> in tournaments is the whole point…"* *(edge_note: "stats plus qualification history carry argument")*

This is a genuine `analysis` post with a two-sided, load-bearing argument. The catch is that it has **no hard digits** ("nearly a
goal a game" is qualitative). The model learned `analysis` to be something associated with  a stat block, so a confident
opening with no numbers reads as an opinion → `hot_take`. The missed boundary: **load-bearing
reasoning without explicit metrics is still analysis** 

**Error 2 — `analysis → hot_take` (post 48, confidence 0.39)**
> *"the obsession with assists as THE playmaking stat is so off. Iniesta and Xavi created entire
> tournaments worth of chances from deep…"* *(edge_note: "named players carry the tactical point")*

Same boundary, but worse. In this post, there is a strong assertion ("is so off") plus evidence that represents **named comparison**, not statistics. As such the model treats "no stat + assertive tone" as `hot_take`. This is the "naming players isn't evidence" rule biting in reverse: the model over-applied it where the named reasoning genuinely *does* carry the argument.

**Error 3 — `reaction → hot_take` (post 178, confidence 0.41)**
> *"uruguay are defending like their lives depend on it and honestly it might, because my blood
> pressure is through the absolute roof"*

This post contains pure in-match emotion making it a `reaction`, but it leads with a **declarative, confident clause**. I presume the model recognizes these parts as `hot_take` and therefore makes the mistake. This is the cosliest error type since I flagged the `hot_take ↔ reaction` boundary the most. The model seems to have not learned the reasoning about protability in this case.

### 6.3 Labeling problem or data/prompt problem?

**It's a data-distribution + boundary-subtlety problem, not annotation inconsistency.** All three
misclassified `analysis` posts are labeled *consistently* with the refined definition, with matching
`edge_notes` reasoning. The model fails because:

1. **`analysis` is the smallest class (55 posts), mostly with explicit metrics** — so the model learned
   *"analysis = long post with numbers"* instead of *"conclusion carried by load-bearing evidence."*
2. **`hot_take` is the plurality class (40%)** — the natural low-confidence fallback.
3. **The `hot_take ↔ reaction` line is semantic (portability), not lexical** — no surface token marks
   it, so a feature-driven model has nothing to latch onto.

Independent corroboration: **the baseline LLM — which only read the rubric — got the same `analysis`
recall of 0.62 (S5.4).** Two different systems missing the same posts signals the problem lives in the
*examples* (scarce, surface-ambiguous qualitative-evidence cases), not in inconsistent labeling.

### 6.4 What would fix it

- **More `analysis` examples with qualitative / named-comparison evidence** (not stat blocks) — to
  break the "analysis = digits" shortcut (posts 36, 48, 120).
- **More event-bound `reaction` examples with confident declarative clauses** — to teach the
  portability boundary (posts 08, 178).
- **Class weighting / rebalancing** so `hot_take` stops being the uncertainty default.
- The *definitions* don't need tightening — the gap is **example coverage of the hard cases**, not the
  rubric.

---

## 7. Sample Classifications (fine-tuned model)

Real outputs from the fine-tuned model, each with predicted label and confidence. The first four are
error cases that illustrate the `hot_take`-sink behavior.

| # | Post (excerpt) | True | Predicted | Confidence | Correct? |
|---|---|---|---|:---:|:---:|
| 1 | "Bouaddi is seriously impressive. … seemed like the players didn't believe they were getting a goal…" | `reaction` | `hot_take` | 0.35 | ❌ |
| 2 | "Haaland's club numbers are insane … club scoring and tournament pedigree are two different things…" | `analysis` | `hot_take` | 0.36 | ❌ |
| 3 | "Özil racked up 19 Premier League assists in 2015-16 to match Henry's record and still got scapegoated…" | `analysis` | `hot_take` | 0.35 | ❌ |
| 4 | "uruguay are defending like their lives depend on it … my blood pressure is through the absolute roof" | `reaction` | `hot_take` | 0.41 | ❌ |
| 5 | "Hot take but Mbappé is genuinely the most overrated guy at this whole tournament. France would be way more balanced if he wasn't hoovering up every bit of the attack…" (post 16) | `hot_take` | `hot_take` | 0.87| ✅ |

**Why this prediction is reasonable (post 16):** A *textbook example* of a `hot take` prototype can be distilled into one clear statement about an athlete with no quantifiable proof behind that statement, yet able to withstand at least a week's time before backing that up, starting off the statement with `Hot Take`. This is also the area where the model most successfully learned about this specific category of the data set (`hot take` recall 13/13), so therefore having an accurate and confident prediction reflects a *legitimately learned signal* versus low-confidence (<.5) predictions on the errors created by the other 4 predictions listed above.

---

## 8. Did it hit the success criteria?

The planning doc pre-registered three measurable targets ([planning.md S6](planning.md)):

| Target | Result | Met? |
|---|---|:---:|
| Macro-F1 ≥ 0.80 | **0.832** | ✅ |
| Per-class recall ≥ 0.75 for `analysis` | **0.625** | ❌ |
| `reaction → hot_take` ≤ 15% of the reaction row | **2 / 10 = 20%** | ❌ |

The overall performance of the model was adequate with respect to the macro-f1. However, it failed to satisfy both community-driven criteria - on both of those axes (per the planning document). Specifically, the analysis recall fell short (qualitatively, analysis leaks into hot take), and the biggest mistake (reaction --> hot take) was over ceiling. Per the definition in the project, this model is not yet "good enough" for auto-tagging and can only be used with low confidence (~<0.4) predictions that will be checked by a human reviewer. Human review would correct all 5 of the errors committed by this model. All of the errors were with low confidence.

---

## 9. Reflection — what the model captured vs. what I intended

I successfully set up an `analysis` to mean "*A conclusion supported by load-bearing evidence — namely statistics, records, **and** structured named logical reasoning*." The model learned a different proxy for this class — that is, anyway **"An analysis is an extremely long posting with numerous statistics."** It had good precision by nailing the core concept (the core portion of the wide digested posts), but missed the qualitative / short tails of posts completely (i.e. post #36 — vice post #48; post #120).

I meant for a `hot_take` to mean ***"An assertion with no evidence that is transportable,"*** but the model over-analyzed this class of posts, causing it to become the defacto bucket of uncertainty. By over-analyzing, the model developed full recall for this class and included anything that had an assertive surface structure (that is, a surface structure) in reaction or non-numeric analysis. 

**The Key Line:** **the model overfitted to the surfaces of the postings on the length of the posting, the number of digits in the posting, or the level of confidence in the posting** (i.e. surface test characteristics of postings) instead of the semantic tests to be defined by me (i.e. ***Is the Evidence load bearing and does it Have Support?***, ***Is the Claim Transportable?***). The model essentially failed along the boundary defined by my formalator (i.e. `hot_take ↔ reaction`); as such confirming that boundary would be semantic and not lexical. The difference between the definitions developed by me and the learned `hot_take - reaction` boundary is also the difference between: *Intent* (that is why that posting would be considered as Evidence) and *Form* (that is what typical Evidence would look like).

---

## 10. Spec Reflection

*How the specification helped:* Creating a decision rule for `hot_take ↔ reaction` and an 
`edge_notes` audit column **prior** to data collection (planning S3, S7) resulted in a consistent 
annotation of `hot_take ↔ reaction` and a **pre-registered hypothesis** of confusion between `hot_take ↔ 
reaction`. The evaluation of the data then **tested** that hypothesis which was confirmed by the 
confusion matrix.

*How implementation diverged from specification:* The original definition of 
`hot_take` in the specification was limited to **players/teams/managers** only; however, 
there were posts that did not fit that criteria. One example of this was post 03 
where the user shared their feelings about the problems with 
**hydration breaks** as a **confident claim about the game itself**. So, while I was 
starting the **annotation** of the data, I changed my definitions after I had started 
the project (planning S8) to broaden the definition of `hot_take` to also include 
rules/format/officiating and re-keyed the distinction between "player vs event" to 
"portable vs expiring." I also chose to leave the **natural** 55/82/69 imbalance in 
the data (as documented in the specification) instead of creating `analysis` examples 
which was, in part, related to the `hot_take`-sink behavior.

---

## 11. AI Usage

**1. Testing stress labels for various classified boundaries(S7A).** I created an AI generated set of historical label boundaries for posts and then created and tried to categorize examples I could see that were difficult to categorize. Examples included unqualified managerial declarations of "hot take" style. I chose to override the AI's classification of these examples and created a new set of labels for those examples, specifically; the "hot take" label was increased in its availability and the previously categorized attribute "Player vs Event" changed to "Portable vs Expiring". This provided me with the ability to identify many of the ambiguous examples as well as make the final determination on a newly defined taxonomy.

**2. Collection of error data for subsequent reporting (this session).** I have grouped the five noted errors into representative patterns; computed the metrics / confusion matrix from the initial counting of the noted errors; and confirmed all noted patterns to the initial post text. Of note was the removal of the hypothesis "sarcasm drives the noted error" after revisiting the actual text of all five examples; because there was no sarcasm in any of the examples. The noted "Haaland" mischaracterization occurred in an incorrect quadrant; I have confirmed the correct location of the example (36) and have supported the original 0.35 - 0.41 attribution based on the 0.35 - 0.41 coefficients.

**3. Annotation help (planning S7B).** The large languange model produced pre-labels based solely on ambiguous characteristics for each example, rather than for the entire subset of unique characteristics for all examples. Each pre-label that was generated underwent human review for coding assignment. Therefore, the ultimate label assignment for each example occurred at the hands of humans, with the supporting original coding label annotated in the accompanying edge notes.
