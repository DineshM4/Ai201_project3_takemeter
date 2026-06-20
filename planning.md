# Project 3 — Text Classification Planning

## 1. Community

**Community chosen:** **r/football** (~16M subscribers) — a general association football
subreddit covering the Premier League, La Liga, Champions League, World Cup, and beyond.

**Why this community:**
- Text posts in this subreddit have a wide quality of content, from full breakdown of matches, to instant emotional dumps after results  
- High volume and public accessibility make balanced data collection possible.

**Why it fits a classification task:**
- Multiple distinct post *intents* coexist in one feed. These include reasoned argument, confident
  opinion, and raw emotional response, each of which I believe can be split into labels.
- The community polices these distinctions itself. For example, pushback phrases like *"where's the
  evidence,"* *"another hot take,"* and *"based on what?"* reveal that members already psuedo-sort classifications in this subreddit
- Because users sort through entertainment quality from r/football, it is valuable for use of a classifier in classifying hot takes and emotional responses. Reactions or hot takes will be seen as two sides of the same coin, yet will have different evidentiary expectations; reactions are based on emotion, hot takes imply factual basis. A model that would confuse them and rank hot takes underneath factual evidence would be inaccurate to how the community communicates.

---

## 2. Labels

Three labels: **analysis**, **hot_take**, **reaction**.

### Label A — `analysis`
**Definition:** This post contains an argument with evidence included in the argument therefore removing all statistics, tactical observations with specific players as examples, or historical comparisons to the argument removes all proof for the statement.

- **Example 1:** _"Under Slot, Liverpool's PPDA sits at 4.8 — 3rd in the PL and higher
  intensity than Klopp's last full season (6.2). The press isn't weaker, it's actually more
  aggressive, just more zonal. Media narrative is wrong."_
- **Example 2:** _"Lewandowski's UCL knockout conversion rate (24%) vs Ronaldo's in the same
  rounds (19%) over the last decade — people say Lewandowski choked in Europe but the
  finishing numbers don't back that up."_
- **Uncertain case:** _"Bellingham has the highest progressive passes per 90 among Real
  Madrid midfielders (7.2), but his defensive output is weak — is the hype real?"_ — The post has one
  specific stat cited, but the conclusion ("is the hype real?") itself is vague and open-ended.
  I have the post leaning toward **analysis** because the evidence is explicit and verifiable, but the lack of a structured argument makes it thin.

### Label B — `hot_take`
**Definition:** This post includes a bold confident claim about a player/team/manager without any evidence to back that up. Therefore, the claim itself is the evidence.

- **Example 1:** _"Mbappe has never been a big-game player and never will be. Champions
  League exposes him every time."_
- **Example 2:** _"Ten Hag was a fraud at United. That squad wasn't as bad as people say.
  Any top manager would have gotten more out of them."_
- **Uncertain case:** _"Watching Pedri over three seasons, it's clear he's the best
  midfielder of his generation — his ability to find space is just different."_ — The poster
  implies informed opinion based on watching, but no specific evidence is provided. I have the post leaning toward **hot_take** because the absence of specificity means the argument is an assertion.

### Label C — `reaction`
**Definition:** This post is an emotional reaction to a recent event (game score/gol/transfer announcement), and therefore there is no way to separate the emotion from the response and without the event the post would not exist.

- **Example 1:** _"THAT MBAPPE GOAL. The angle from that distance? How is that even
  physically possible??"_
- **Example 2:** _"Up 2-0 with 15 minutes left and we still lost 3-2. Same script every
  season. I'm done."_
- **Uncertain case:** _"We lost 4-0 today and it proves what's wrong with this manager: no
  press, no shape, no plan. He needs to go."_ — At first glance, this looks like a hot_take (general claim
  about a manager), but the post only exists because the match just ended and the poster is
  upset. As such, I have the post Leaning toward **reaction** because the event trigger drives everything because the opinion wouldn't exist without the result.

---

## 3. Hard Edge Cases

**Mutual exclusivity check**

| Pair | Boundary |
|------|----------|
| analysis vs hot_take | Presence of specific, verifiable evidence. Strip it: analysis becomes hot_take. |
| hot_take vs reaction | Event-bound vs. general. Hot takes make claims about players/teams across time; reactions are triggered by what just happened. |
| analysis vs reaction | These rarely overlap — analytical posts are prepared, not triggered by a 90th-minute result. |

**The hardest pair — hot_take vs reaction:** a match result prompts a *general managerial
claim* (e.g. "we lost 4-0 today and it proves the manager has no plan, he needs to go"). The
general claim reads like a hot_take, but the post is event-bound.

**Decision rule:** If the post is explicitly about a specific event that just happened **and** the opinion is a consequence of that event (not the reason for posting), label it **reaction**. Diagnostic question applied to every borderline post: *if you remove the recent event, does the post still exist?* If yes → **hot_take**; if no → **reaction**.

**Where I'll record these:** an `edge_notes` column in the dataset capturing which rule fired
for each borderline post, so I can audit annotation consistency later.

---

## 4. Data Collection Plan

> _Where will you collect examples? How many per label? What if a label is
> underrepresented after 200 examples?_

**Source(s):** r/football text posts via `manual export`.

**Target dataset size:** `[~200]` examples, roughly balanced across the three labels.

| Label | Target count |
|-------|--------------|
| analysis  | `[~67]` |
| hot_take  | `[~67]` |
| reaction  | `[~67]` |

**Collection method:** Sample broadly rather than cherry-pick so the set isn't skewed toward
one type — e.g. recent-result threads over-index reactions, while older threads surface more
analysis. 

**Expected imbalance risk:** `analysis` is likely the rarest class since it takes effort to write,
while `reaction` spikes around match days — note this so collection windows don't bias the set. This bias will be even more noticable since this project is being completed at the time of the world cup.

**If a label is underrepresented after 200 examples:**
- Targeted search: look within contexts where the specific class you want to find is concentrated (for example, for `analysis`, only pull threads that have either fbref/StatsBomb links or are weekly tactical threads; for `reaction`, only use threads that are post-match and were created within hours of when the game ended). 
- Fallback: if you can't find any examples of a genuinely rare class, leave the natural imbalance in your dataset and document it so you can deal with it in your evaluation (i.e., report metrics for each class, use macro-F1, consider weighting the classes) instead of trying to create examples for that class.

---

## 5. Evaluation Metrics

> _Which metrics, and why are they right for THIS task? Accuracy alone is not enough —
> explain what else you need and why._

**Primary metrics:**
- **Per-class precision, recall, and F1** — catches a model that scores well overall but
  fails on the rarer `analysis` class, which is the class the community values most.
- **Macro-averaged F1** — weights each label equally so the easier/more-frequent classes
  (`reaction`, `hot_take`) can't mask weak performance on `analysis`.
- **Confusion matrix** — shows *which* labels get confused. This is directly tied to the
  Section 3 edge case: I specifically expect hot_take ↔ reaction confusion, and the matrix
  is how I'll measure whether my decision rule actually holds up in practice.

**Why accuracy alone is insufficient here:**
Classes are usually not totally balanced (analysis happens less often; reactions are high around the day of the contest), but the most difficult distinction (hot_take vs. reaction) uses two of the largest classes. A model could be highly accurate by combining the reaction class with the hot_take class and would still totally fail to create a distinction that helps the community. The confusion matrix and per class recall will expose this distinction.

**Which errors matter most:** A reaction that is misclassified as a hot_take will create the most expensive error (the failure mentioned in Section 1, where emotional, momentary posts are treated as a reasonable position without supporting evidence). Therefore, I will watch the confusion matrix at the reaction→hot_take cell the most.

---

## 6. Definition of Success

> _What performance makes this classifier genuinely useful? What's "good enough" for
> deployment in a real community tool? Make the criteria objective and measurable._

**Concrete, measurable targets:**
- Macro-F1 ≥ `[0.80]` on a held-out test set of `[~40]` examples.
- Per-class recall ≥ `[0.75]` for `analysis` (the class the community values most).
- The reaction→hot_take confusion-matrix cell ≤ `[15%]` of the reaction row (the costliest error).

**Baseline to beat:** majority-class baseline ≈ `[~33% accuracy with balanced classes]`; the
model must clearly exceed this. (Also worth beating: a simple keyword/TF-IDF baseline.)

**"Good enough" for deployment:** accurate enough to auto-tag or sort posts by discourse type
in a community tool, with low-confidence predictions surfaced for human review rather than
auto-applied. The reaction↔hot_take distinction in particular must be reliable, since that's
the misread that would most misrepresent the community.

**Self-check:** _Are these criteria specific enough that, at the end, I can objectively
answer yes/no on whether I hit them?_ `[Yes — each target above is a number measured on
a defined test set. / Note any criterion still too vague and tighten it.]`

---

## 7. AI Tool Plan

> _This is a data/annotation project, not an implementation project — there's no code to
> generate. AI tools help in three specific places below._

### 7a. Label Stress-Testing 
- **Plan:** Give the AI my Section 2 label definitions and Section 3 edge case, and ask it
  to generate 5–10 posts that sit on the boundary between two labels.
- **What I'm looking for:** posts I *cannot* classify cleanly with my current definitions.
- **Action:** Every such post means my definitions are too loose — I will tighten the
  definitions and/or decision rules in Sections 2–3 _before_ collecting data.
- **Tool:** `Claude`
- **Status / notes:** record which definitions I changed after this exercise:
  - broadened `hot_take` to also cover **an aspect of the game/its rules** (not just player/team/manager), and clarified that reasoned-but-unsupported arguments count as `hot_take`, not `analysis`.:
  - re-keyed the `hot_take` ↔ `reaction` boundary from *event-bound* to *venting vs. portable claim* (since during the World Cup nearly everything is event-triggered): `reaction` = expiring emotion; `hot_take` = a claim the poster would still defend a week later. Diagnostic: strip the emotion and event — if a defensible claim remains → `hot_take`, else → `reaction`.

### 7b. Annotation Assistance
- **Will I use an LLM to pre-label?** Yes, but only when I cannot differentiate the label of the post
- **Tool:** Claude
- **Tracking:** Add all special cases to `edge notes`


### 7c. Failure Analysis
- **Plan:** Export all wrong predictions (text + true + predicted) and ask Claude to group them into patterns.
- **What I'm looking for:** reaction↔hot_take confusion, length/sarcasm effects, thin-analysis cases, World Cup match-day skew.
- **How I'll verify:** for each pattern, pull the actual errors, confirm it holds, and quantify it before writing it up.
- **Tool:** Claude.

---

## 8. Refined Definitions (from the labeled data)

> These supersede the Section 2 definitions. They are rewritten from what actually
> separated the 206 labeled posts (55 `analysis`, 82 `hot_take`, 69 `reaction`) and the
> `edge_notes` recorded while annotating. The biggest shifts: `hot_take` now covers the
> game itself (rules/format/officiating), `analysis` accepts non-statistical *logical*
> evidence, and the `hot_take`↔`reaction` line is **portable claim vs. expiring emotion**,
> not player-vs-event.

### `analysis`
A post whose conclusion is **carried by verifiable, load-bearing evidence** — remove the
evidence and the claim collapses. Evidence qualifies as any of:
- **Statistics / metrics** (PPDA, xG, conversion %, pass completion) — *post 04, 84*.
- **Historical comparisons or records** (named tournaments, scorelines, Ballon d'Or counts,
  all-time records) — *post 14, 21, 45, 126*.
- **Structured logical evidence with named specifics**, even without a stat, when the
  reasoning itself is the proof (e.g. "10 corners and no scoring threat → we need better
  finishers") — *post 15, 48*.

Naming players is **not** evidence on its own (*post 07*). The post must be prepared and
argued, not triggered by a result.

### `hot_take`
A **bold, confident, portable claim** asserted **without verifiable evidence** — the claim
is offered as its own justification. Covers claims about:
- **players / teams / managers** (*post 16, 22, 37*), **and**
- **the game itself**: rules, format, officiating, equipment, commercialization
  (*post 03, 11, 34, 49, 70, 196*) — this was added after annotation; the original
  definition only covered people.

A claim is `hot_take` (not `reaction`) if it is **portable** — the poster would still defend
it a week later, independent of any single match (*post 19, 46, 205: "borderline reaction
but claim is portable"*). Reasoned-but-unsupported arguments and rhetorical questions that
carry an implied unsupported claim are `hot_take`, not `analysis` (*post 13, 28, 136*).

### `reaction`
A post driven by **expiring, event-bound emotion** — it exists only because something just
happened (a goal, scoreline, red card, ref decision, substitution) and the emotion cannot be
separated from that moment. Includes in-match running commentary, live tactical reads tied to
an ongoing game, shock at a scoreline, and ref/VAR frustration bound to *this* match
(*post 17, 20, 44, 110, 119*). Strip the event and the emotion and **nothing defensible
remains**.

**Decision rule (the hard `hot_take`↔`reaction` boundary):** strip the emotion and the
specific event. If a claim you'd still defend a week later remains → `hot_take`. If only an
expiring feeling remains → `reaction`. (Restated from 7a; this is the rule the `edge_notes`
actually applied.)

| If the post… | Label |
|---|---|
| stands on verifiable stats, records, or load-bearing named reasoning | `analysis` |
| asserts a portable, evidence-free claim about people **or** the game | `hot_take` |
| only exists because of a just-happened event and carries no portable claim | `reaction` |

---
