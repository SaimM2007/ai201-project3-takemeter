# TakeMeter - Planning Document
## AI201 Project 3

---

## Community

I chose r/nba because it's one of the most active sports communities on Reddit and the discourse is all over the place in terms of quality. You'll see actual stat breakdowns right next to someone yelling that a player is washed with zero evidence. The community genuinely cares about the difference between a good take and a bad one, people call each other out for bad reasoning all the time. Posts are short, text-heavy, and fall into pretty recognizable patterns which makes it a good fit for a classification task.

---

## Label Taxonomy

### Labels

| Label | Definition |
|-------|------------|
| `analysis` | The post makes a structured argument backed by statistics, historical comparison, or tactical observation. Evidence is specific and the post is reasoning toward a conclusion. |
| `hot_take` | A bold confident opinion stated without substantial supporting evidence. The claim might be true but the post is asserting rather than arguing. Includes overreactions and controversy bait. |
| `hype` | An immediate emotional reaction to a specific play, moment, or event. Little to no argument, the post is just expressing excitement, celebration, or disbelief. Includes highlights and throwbacks. |
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

Post: "LeBron is overrated - his playoff win rate against top-seeded opponents is below .500."

Could be `analysis` (cites a stat) or `hot_take` (the framing is accusatory and the stat is cherry-picked).

Decision rule: If the evidence is being used to genuinely reason through a claim with multiple data points and context, label it `analysis`. If the stat is just there to dress up an assertion and the post reads as conclusion first, evidence second, label it `hot_take`. This one goes `hot_take` because the framing is accusatory and it's one cherry-picked stat not an actual argument.

**Edge Case 2: News post with commentary**

Post: "[ESPN] Giannis traded to Heat: Grades, reaction, Bucks' next steps - Miami Heat B-, Milwaukee Bucks B+"

Has sourced reporting but also grades and editorial takes mixed in.

Decision rule: If the post title starts with a journalist tag ([Reporter Name]) or has a factual announcement as the primary content, it's `news` even if grades or reactions follow. The core event is the news, everything else is commentary. Goes `news`.

**Edge Case 3: Question that includes analysis**

Post: "Has modern NBA parity changed how we should evaluate all time great players?"

Asks a question but frames it with genuine reasoning about the premise.

Decision rule: If the post ends in a question mark and is primarily soliciting community input rather than making an argument, label it `question` regardless of how substantive the setup is. Goes `question`.

**Edge Case 4: Positive hot take that sounds like hype**

Post: "Wembanyama is already better than any big man the NBA has seen since prime Shaq"

Bold claim (hot_take) but sounds like excitement about a player (hype).

Decision rule: If the post is making a comparative claim or ranking assertion, even an enthusiastic one, it's `hot_take`. Hype is for reactions to specific events or moments, not general claims about player quality. Goes `hot_take`.

---

## Why These Labels Matter

These distinctions map to real norms in r/nba. People there actively reward posts that bring receipts (analysis) and call out posts that are just loud opinions with nothing behind them (hot_take). Reaction posts to big plays (hype) and breaking news (news) are their own distinct thing. Discussion threads (question) exist everywhere. The community uses these exact concepts informally all the time which is why they're the right things to measure.

---

## Data Collection Plan

- **Source:** r/nba hot feed, scraped June 2026
- **Method:** Public Reddit JSON API (no authentication needed), filtered for non-stickied posts with title length over 20 characters. Supplemented with manual collection from multiple scroll sessions.
- **Target:** around 40 examples per label (200 total)
- **If underrepresented:** If any label falls below 30 examples after initial collection, manually collect more for that label by browsing r/nba filtered by flair or searching for known patterns like "[Charania]" for news posts.

### Label Distribution (Final)

| Label | Count | % |
|-------|-------|---|
| analysis | 43 | 21.5% |
| hot_take | 39 | 19.5% |
| hype | 42 | 21.0% |
| news | 40 | 20.0% |
| question | 36 | 18.0% |

Ended up pretty balanced, all labels within 3.5% of each other.

---

## Evaluation Metrics

**Overall accuracy** is the primary metric since the classes are balanced. With 5 balanced classes random guessing gets 20% so anything meaningfully above that shows the model actually learned something.

**Why accuracy alone isn't enough:** A model could score decent accuracy by doing well on 3-4 labels while completely failing on 1-2. Per-class F1 catches that.

**Per-class F1** is the most useful per-label number. It penalizes both over-predicting a label (low precision) and missing real examples of it (low recall). This is especially important for `hot_take` and `analysis` where the boundary is fuzzy. If those two have bad F1 specifically it means the model couldn't learn the hardest distinction in the dataset.

**Confusion matrix** shows which labels are getting confused and in which direction. A systematic pattern like analysis consistently predicted as hot_take tells you way more than individual errors because it points to a specific boundary the model failed to learn.

---

## Definition of Success

For a real community tool the classifier would need at least 0.70 overall accuracy and per-class F1 above 0.60 for all labels. Below that users would notice wrong labels too often and stop trusting it.

For this class project given the small dataset (200 examples): 0.55+ accuracy with no label having F1 of 0.00 is acceptable. The model should have learned something for every class. A model that collapses everything into one label is not acceptable regardless of overall accuracy.

---

## AI Tool Plan

**Label stress-testing:** Before annotating I gave Claude the label definitions and asked it to generate 10 posts sitting at the `analysis` vs `hot_take` boundary. Several were genuinely hard to classify which pushed me to add the "conclusion first, evidence second" decision rule for `hot_take` before touching any data.

**Annotation assistance:** Claude was used to pre-label the 200 posts after being given the full label definitions. Every pre-assigned label was reviewed manually. Around 15-20% got corrected during review, mostly `analysis` vs `hot_take` borderline cases and positive `hot_take` posts that got pre-labeled as `hype`.

**Failure analysis:** After running the notebook the wrong predictions were reviewed to find patterns. The main finding was that the fine-tuned model collapsed `analysis`, `hot_take`, and `news` almost entirely into `hype`. Confirmed by reading the confusion matrix directly since 14 of 19 errors were `hype` predictions on non-hype true labels.