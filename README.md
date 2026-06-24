# TakeMeter - r/nba Discourse Classifier
### AI201 - Project 3

A fine-tuned text classifier that categorizes r/nba posts into 5 types: analysis, hot take, hype, news, or question.

---

## Community

r/nba is one of the biggest sports communities on Reddit. Posts range from breaking trade news to stat breakdowns to pure reaction posts after a big play. People in the community actually care about the difference between a well-reasoned take and someone just yelling into the void, so these labels mean something to real users there. Almost every post fits cleanly into one of a few patterns which makes it a good fit for classification.

---

## Label Taxonomy

| Label | Definition |
|-------|------------|
| `analysis` | Structured argument backed by stats, historical comparison, or tactical observation |
| `hot_take` | Bold confident opinion with little to no supporting evidence |
| `hype` | Immediate emotional reaction to a play, moment, or event (highlights, throwbacks, celebrations) |
| `news` | Factual report from a journalist or insider: trades, signings, injuries, front office moves |
| `question` | Post asking the community for opinions, predictions, or information |

---

## Dataset

- **Source:** r/nba hot feed, June 2026
- **Size:** 200 posts
- **Collection:** Public Reddit JSON API + manual collection
- **Split:** 140 train / 30 val / 30 test (stratified)

### Label Distribution

| Label | Count | % |
|-------|-------|---|
| analysis | 43 | 21.5% |
| hot_take | 39 | 19.5% |
| hype | 42 | 21.0% |
| news | 40 | 20.0% |
| question | 36 | 18.0% |

Pretty balanced across all labels, all within 3.5% of each other so no majority class bias issues.

### Labeling Process

Each post was labeled by reading the title and applying the definitions from planning.md. The main question was: does this post reason with evidence (analysis), assert without evidence (hot_take), react to a moment (hype), report a fact (news), or ask something (question)?

### Hard Cases

1. **"[ESPN] Giannis traded to Heat: Grades, reaction, Bucks' next steps - Miami B-, Bucks B+"** - has both reporting and editorial grades in it. Labeled `news` because the core content is a sourced factual event, the grades are just secondary commentary.

2. **"Has modern NBA parity changed how we should evaluate all time great players?"** - frames a thoughtful premise but is really just asking the community. Labeled `question` because it's soliciting responses not making an argument.

3. **"Giannis Trade Overreactions - r/nba is acting like Giannis is some washed-up, hobbled old man"** - pushes back on community reaction but doesn't cite any real evidence. Labeled `hot_take` because it's asserting a position without data to back it up.

---

## Model

- **Base model:** `distilbert-base-uncased` (66M parameters)
- **Fine-tuning:** Added a 5-class classification head, trained for 3 epochs on 140 examples with validation-based checkpointing
- **Key hyperparameter decision:** Kept the default learning rate of 2e-5. With only 140 training examples a higher rate would risk unstable updates. Batch size of 16 fit the T4 GPU fine.
- **Other hyperparameters:** weight decay 0.01, warmup steps 50, eval per epoch

---

## Evaluation Report

### Accuracy

| Model | Accuracy |
|-------|----------|
| Zero-shot baseline (Groq llama-3.3-70b-versatile) | 0.733 |
| Fine-tuned DistilBERT | 0.467 |
| Difference | -0.267 (regression) |

The fine-tuned model actually did worse than the zero-shot baseline. More on why in the reflection section.

### Per-Class Metrics - Fine-Tuned Model

```
              precision    recall  f1-score   support

    analysis       0.50      0.14      0.22         7
    hot_take       0.00      0.00      0.00         6
        hype       0.43      1.00      0.60         6
        news       1.00      0.33      0.50         6
    question       0.56      1.00      0.71         5

   accuracy                           0.47        30
  macro avg       0.50      0.49      0.41        30
weighted avg       0.46      0.47      0.38        30
```

### Per-Class Metrics - Baseline (Groq)

```
              precision    recall  f1-score   support

    analysis       0.75      0.43      0.55         7
    hot_take       1.00      0.67      0.80         6
        hype       1.00      0.67      0.80         6
        news       0.50      1.00      0.67         6
    question       0.83      1.00      0.91         5

   accuracy                           0.73        30
  macro avg       0.82      0.75      0.74        30
weighted avg       0.81      0.73      0.73        30
```

### Confusion Matrix (Fine-Tuned Model)

![Confusion Matrix](confusion_matrix.png)

|  | pred: analysis | pred: hot_take | pred: hype | pred: news | pred: question |
|---|---|---|---|---|---|
| **true: analysis** | 1 | 0 | 5 | 0 | 1 |
| **true: hot_take** | 0 | 0 | 5 | 0 | 1 |
| **true: hype** | 0 | 0 | 6 | 0 | 0 |
| **true: news** | 0 | 0 | 4 | 2 | 0 |
| **true: question** | 0 | 0 | 0 | 0 | 5 |

The model basically just predicted `hype` for almost everything except `question`. It got `question` perfect (5/5) and partially learned `news` (2/6) but completely collapsed `analysis` and `hot_take` into `hype`.

### Error Analysis

**Error 1**
- **Text:** "After the KD Nets were eliminated in 2021, Jackie MacMullan claimed KD's goal was to win 3 championships with Brooklyn."
- **True label:** `analysis`
- **Predicted:** `hype` (confidence: 0.22)
- **Analysis:** This post cites a specific journalist and a specific claim, so it reads like analysis because it's drawing a historical parallel with a source. The model predicted `hype` probably because it picked up on the dramatic framing ("KD Nets eliminated") instead of the argumentative structure. With only 30 training examples for analysis the model never learned to tell the difference between sourced reasoning and excited reaction posts.

**Error 2**
- **Text:** "Wembanyama is already better than any big man the NBA has seen since prime Shaq"
- **True label:** `hot_take`
- **Predicted:** `hype` (confidence: 0.21)
- **Analysis:** Classic hot take: bold claim, no evidence, strong assertion. Model predicted `hype` because the post is celebrating a player in superlative terms which looks like hype on the surface. The line between `hot_take` and `hype` gets blurry when the hot take is positive instead of controversial since both involve enthusiasm. The model never picked up that one is making a claim and the other is just reacting to a moment.

**Error 3**
- **Text:** "[Krawczynski] Timberwolves have held discussions on Giannis, Kyrie, Trey Murphy III, Josh Giddey, Derrick White"
- **True label:** `news`
- **Predicted:** `hype` (confidence: 0.21)
- **Analysis:** This one has a journalist tag in brackets which is basically a dead giveaway for `news`, plus it lists specific player names and a trade context. Model still predicted `hype`. It clearly never learned the `[Reporter]` bracket pattern at all, which the Groq baseline got right easily since it already knows what that format means. With only 140 training examples spread across 5 labels there just weren't enough news examples for the pattern to stick.

### Sample Classifications

| Post | Predicted | Confidence | Correct? |
|------|-----------|------------|----------|
| "Who do you think will be the Ajay Mitchell in this year's upcoming draft?" | `question` | 0.81 | Yes |
| "14 Years Ago Today - Chris Bosh Pours Champagne After Winning Championship" | `hype` | 0.74 | Yes |
| "[Charania] BREAKING: Dusty May agreed to become new head coach of Dallas Mavericks" | `hype` | 0.21 | No (true: `news`) |
| "Trae Young will never win a championship. His defense is just too bad." | `hype` | 0.22 | No (true: `hot_take`) |
| "Steph Curry drops 52 on 9-15 from three in a blowout win" | `hype` | 0.79 | Yes |

The `question` prediction makes sense: the post is clearly asking for community input with no stated position and the model picked up on the interrogative structure. The wrong `hype` predictions on `news` and `hot_take` show the collapse problem where the model just defaulted to its strongest learned class.

### Reflection

The model learned two things: `question` (interrogative structure is pretty obvious) and `hype` as basically a catch-all for everything else. It completely failed to learn the subtler distinctions like `analysis` vs `hot_take`, `news` vs `hype`, or the journalist tag pattern for `news`.

The gap between what I wanted the model to learn and what it actually learned is pretty big. I wanted it to pick up on argumentative structure: does this post reason with evidence or just assert something? Instead it learned surface level vocabulary cues and when those were ambiguous it just defaulted to `hype` since that was the dominant pattern in excited sports language.

The main reason this happened is 140 training examples is honestly just too small to learn 5 fine-grained distinctions, especially when 4 of the 5 labels all involve opinionated language that overlaps a lot. The Groq baseline crushed the fine-tuned model (0.733 vs 0.467) because llama-3.3-70b already knows from pretraining what journalist tags mean, what a hot take looks like, and how questions differ from assertions. DistilBERT had to learn all of that from scratch with almost no data.

---

## Spec Reflection

The spec was most helpful in the label design phase. Having to write decision rules for edge cases before annotating forced me to actually think through the `analysis` vs `hot_take` boundary before touching any data. Without that step I probably would have labeled similar posts differently throughout the dataset which would have made the training signal way noisier.

Where I diverged: the spec kind of assumes fine-tuning will improve on the baseline and the whole evaluation section is framed around "fine-tuning improvement." My model regressed by 0.267. I kept the honest numbers instead of tweaking things to get better results because the regression is actually informative. It shows both the data size limitation and how much of this task the Groq baseline handles through world knowledge that DistilBERT had to learn from zero.

---

## AI Usage

1. **Data collection and labeling:** Claude was used to pre-label the 200 posts after being given the label definitions from planning.md. Every label was reviewed manually and corrected where needed. Around 15-20% of pre-assigned labels got changed during review, mostly on `analysis` vs `hot_take` borderline cases.

2. **README and planning.md drafting:** Claude generated initial drafts of both documents using the label taxonomy, dataset stats, and notebook output. The error analysis, reflection, and hard case decisions were written and verified by hand using the actual notebook output.

3. **Error pattern analysis:** The wrong predictions from the notebook were reviewed to find the `hype`-collapse pattern described above. The pattern was confirmed by reading the confusion matrix directly.

---

## Files

| File | Description |
|------|-------------|
| `nba_posts.csv` | Labeled dataset (200 examples) |
| `takemeter.ipynb` | Fine-tuning + evaluation notebook |
| `planning.md` | Label taxonomy, edge cases, data collection notes |
| `evaluation_results.json` | Accuracy metrics (generated by notebook) |
| `confusion_matrix.png` | Confusion matrix visualization (generated by notebook) |

---

## How to Run

```bash
pip install transformers datasets scikit-learn torch groq
```

1. Put `nba_posts.csv` and `takemeter.ipynb` in the same folder
2. Open in VSCode or Google Colab
3. Add your Groq API key via Colab Secrets (key name: `GROQ_API_KEY`) or paste directly into the notebook cell
4. Run all cells top to bottom