# TakeMeter — Planning Document
## AI201 Project 3

---

## Community

**r/nba** — the NBA subreddit, one of the most active sports communities on Reddit with millions of members. Discourse ranges from breaking trade news to statistical deep-dives to pure reaction posts after a big play. The community has strong norms around what counts as a "good take" vs. noise, making it a natural fit for discourse quality classification. Posts are short, text-heavy, and fall into recognizable patterns that regular members can identify on sight — which means labels grounded in those patterns should be learnable.

---

## Label Taxonomy

### Labels

| Label | Definition |
|-------|------------|
| `analysis` | The post makes a structured argument backed by statistics, historical comparison, or tactical observation. Evidence is specific and the post is reasoning toward a conclusion. |
| `hot_take` | A bold, confident opinion stated without substantial supporting evidence. The claim might be true, but the post asserts rather than argues. Includes overreactions and media controversy. |
| `hype` | An immediate emotional reaction to a specific play, moment, or event. Little to no argument — the post is expressing excitement, celebration, or disbelief. Includes highlight posts and throwbacks. |
| `news` | A factual report of a real-world event sourced from journalists, insiders, or official announcements. Trades, signings, injuries, front office moves, reporter intel. |
| `question` | The post is asking the community for opinions, predictions, or information. Ends in a question mark or is clearly soliciting responses. |

---

### Examples Per Label

**analysis**
- "Since 1989, there are only 5 teams in NBA history that have had 3 All-NBA players on them that year." (statistical breakdown with historical comparison)
- "During this year's playoffs, Victor Wembanyama recorded 77 defensive stops. No player has recorded more in a single playoff run since the 2010-11 season." (specific verifiable stat with context)

**hot_take**
- "Trae Young will never win a championship. His defense is just too bad." (confident claim, no evidence)
- "The Celtics blew it by not getting Giannis. They are done competing for the next 3 years." (overreaction framed as fact)

**hype**
- "Steph Curry drops 52 on 9-15 from three in a regular season blowout" (pure reaction to a moment)
- "14 Years Ago Today - Chris Bosh Pours Champagne All Over His Face After Winning Championship" (throwback celebration)

**news**
- "[Charania] BREAKING: Dusty May has agreed to become the new head coach of the Dallas Mavericks." (reporter sourced, factual)
- "[Nehm & Amick] League sources say there is a great deal of interest around the league in Herro" (insider intel)

**question**
- "Who do you think will be the Ajay Mitchell in this year's upcoming draft?" (soliciting community opinions)
- "What shot creating guards should the Heat go for if Powell doesn't resign?" (asking for recommendations)

---

### Hard Edge Cases

**Edge Case 1: Stat post with an opinion framing**

Post: "LeBron is overrated — his playoff win rate against top-seeded opponents is below .500."

Could be `analysis` (cites a stat) or `hot_take` (the framing is accusatory and the stat is cherry-picked).

**Decision rule:** If the evidence is being used to genuinely reason through a claim — multiple data points, context, counterarguments considered — label it `analysis`. If the stat exists mainly to dress up an assertion and the post reads as a conclusion first, evidence second, label it `hot_take`. This post → `hot_take`.

**Edge Case 2: News post with commentary**

Post: "[ESPN] Giannis traded to Heat: Grades, reaction, Bucks' next steps — Miami Heat B-, Milwaukee Bucks B+"

This includes sourced reporting but also grades and editorial takes.

**Decision rule:** If the post title starts with a journalist tag ([Reporter Name]) or contains a factual announcement as the primary content, it's `news` even if grades or reactions follow. → `news`.

**Edge Case 3: Question that includes analysis**

Post: "Has modern NBA parity changed how we should evaluate all time great players?"

This asks a question but frames it with genuine reasoning about the premise.

**Decision rule:** If the post ends in a question mark and is primarily soliciting community input rather than presenting a position, label it `question` regardless of how substantive the framing is. → `question`.

**Edge Case 4: Positive hot take that sounds like hype**

Post: "Wembanyama is already better than any big man the NBA has seen since prime Shaq"

This is a bold claim (hot_take) but reads like excitement about a player (hype).

**Decision rule:** If the post is making a comparative claim or ranking assertion — even an enthusiastic one — it's `hot_take`. Hype is reserved for reactions to specific events or moments, not general assertions about player quality. → `hot_take`.

---

## Why These Labels Matter

These distinctions reflect real discourse norms in r/nba. Regular community members actively distinguish between posts that bring receipts (analysis), posts that are just loud opinions (hot_take), posts reacting to a moment (hype), official news (news), and discussion threads (question). The community even has informal norms around flagging bad-faith takes vs. well-reasoned arguments, making these categories meaningful to actual participants.

---

## Data Collection Plan

- **Source:** r/nba hot feed, scraped June 2026
- **Method:** Public Reddit JSON API (no authentication required), filtered for non-stickied posts with title length > 20 characters. Supplemented with manually collected posts from multiple scroll sessions.
- **Target:** ~40 examples per label (200 total)
- **If underrepresented:** If any label falls below 30 examples after initial collection, manually collect additional posts for that label specifically by browsing r/nba filtered by flair or searching for known post types (e.g. searching "[Charania]" for news posts).

### Label Distribution (Final)

| Label | Count | % |
|-------|-------|---|
| analysis | 43 | 21.5% |
| hot_take | 39 | 19.5% |
| hype | 42 | 21.0% |
| news | 40 | 20.0% |
| question | 36 | 18.0% |

Distribution is balanced — all labels within 3.5% of each other.

---

## Evaluation Metrics

**Primary metric: overall accuracy** — straightforward measure of how often the model gets it right. With 5 balanced classes, random baseline is 20%, so anything meaningfully above that shows real learning.

**Why accuracy alone is not enough:** With 5 classes, a model could get high accuracy by doing well on 3-4 labels and completely failing on 1-2. Per-class F1 is needed to catch that.

**Per-class F1:** Reports the harmonic mean of precision and recall per label. This is the most useful single number per class — it penalizes both over-predicting a label (low precision) and missing examples of it (low recall). F1 is especially important for labels like `hot_take` and `analysis` where the boundary is fuzzy: poor F1 on those two specifically would indicate the model failed to learn the hardest distinction.

**Confusion matrix:** Shows which labels are being confused and in which direction. A systematic pattern (e.g., analysis consistently predicted as hot_take) tells you more than individual wrong predictions — it reveals a specific boundary the model didn't learn.

---

## Definition of Success

A classifier is genuinely useful for a community moderation or discourse-quality tool if it reaches at least **0.70 overall accuracy** and **per-class F1 above 0.60** for all labels. Below that, the error rate is high enough that users would notice wrong labels frequently, which undermines trust in the tool.

Acceptable for a class project given the small dataset (200 examples): **0.55+ accuracy** with no label having F1 of 0.00 (meaning the model learned something for every class). A model that collapses everything into one class is not acceptable regardless of overall accuracy.

---

## AI Tool Plan

**Label stress-testing:** Before annotating, I gave Claude the label definitions and asked it to generate 10 posts sitting at the `analysis` vs `hot_take` boundary. Several of those posts were genuinely hard to classify, which prompted me to add the "conclusion first, evidence second" decision rule for `hot_take` before starting annotation.

**Annotation assistance:** An LLM was used to pre-label the 200 posts after being given the full label definitions. Every pre-assigned label was reviewed manually. Approximately 15-20% of labels were corrected during review, primarily `analysis` vs `hot_take` borderline cases and positive `hot_take` posts that were initially pre-labeled as `hype`.

**Failure analysis:** After running the notebook, the wrong predictions list was reviewed to identify patterns. The primary finding was that the fine-tuned model collapsed `analysis`, `hot_take`, and `news` almost entirely into `hype`. This pattern was confirmed by reading the confusion matrix directly — 14 of 19 errors were predictions of `hype` for a non-hype true label.