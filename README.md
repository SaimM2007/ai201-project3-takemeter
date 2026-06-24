# TakeMeter — r/nba Discourse Classifier
### AI201 · Project 3

A fine-tuned text classifier that categorizes r/nba posts by discourse type: analysis, hot take, hype, news, or question.

---

## Community

**r/nba** — one of Reddit's largest sports communities. Discourse ranges from breaking trade news to statistical deep-dives to pure hype posts after a big play. These distinctions matter to the community: members actively reward well-reasoned analysis and call out bad-faith hot takes. The variation in post type makes it a strong fit for a classification task — nearly every post falls cleanly into one of a few recognizable discourse modes.

---

## Label Taxonomy

| Label | Definition |
|-------|------------|
| `analysis` | Structured argument backed by stats, historical comparison, or tactical observation |
| `hot_take` | Bold confident opinion stated without substantial supporting evidence |
| `hype` | Immediate emotional reaction to a play, moment, or event — highlights, throwbacks, celebrations |
| `news` | Factual report from journalists or insiders: trades, signings, injuries, front office moves |
| `question` | Post soliciting community opinions, predictions, or information |

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

Distribution is balanced — all labels within 3.5% of each other, so no majority-class bias issue.

### Labeling Process

Posts were labeled by reading the full title and applying the definitions in `planning.md`. The primary signal was: does the post present evidence (analysis), assert without evidence (hot_take), react to a moment (hype), report a fact (news), or ask something (question)?

### Hard Cases

1. **"[ESPN] Giannis traded to Heat: Grades, reaction, Bucks' next steps — Miami B-, Bucks B+"** — Contains both reporting and editorial grades. Labeled `news` because the primary content is a sourced factual event; the grades are secondary commentary.

2. **"Has modern NBA parity changed how we should evaluate all time great players?"** — Frames a thoughtful premise but is ultimately asking the community. Labeled `question` because it solicits responses rather than presents a position.

3. **"Giannis Trade Overreactions — r/nba is acting like Giannis is some washed-up, hobbled old man"** — Pushes back on community reaction without citing specific evidence. Labeled `hot_take` because the post asserts a position rather than arguing it with data.

---

## Model

- **Base model:** `distilbert-base-uncased` (66M parameters)
- **Fine-tuning:** Added a 5-class classification head, trained for 3 epochs on 140 training examples with validation-based checkpointing
- **Key hyperparameter decision:** Kept the default learning rate of 2e-5, which is the standard starting point for fine-tuning BERT-family models. Given the small dataset (140 train examples), a higher rate risked unstable updates. Batch size of 16 fit the T4 GPU comfortably.
- **Other hyperparameters:** weight decay 0.01, warmup steps 50, eval per epoch

---

## Evaluation Report

### Accuracy

| Model | Accuracy |
|-------|----------|
| Zero-shot baseline (Groq llama-3.3-70b-versatile) | 0.733 |
| Fine-tuned DistilBERT | 0.467 |
| Difference | -0.267 (regression) |

The fine-tuned model performed worse than the zero-shot baseline. This is discussed in the reflection below.

### Per-Class Metrics — Fine-Tuned Model

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

### Per-Class Metrics — Baseline (Groq)

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

The model almost exclusively predicts `hype` for everything except `question`. It correctly learned `question` (5/5) and partially learned `news` (2/6), but collapsed `analysis` and `hot_take` almost entirely into `hype`.

### Error Analysis

**Error #1**
- **Text:** "After the KD Nets were eliminated in 2021, Jackie MacMullan claimed KD's goal was to win 3 championships with Brooklyn."
- **True label:** `analysis`
- **Predicted:** `hype` (confidence: 0.22)
- **Analysis:** This post references a specific journalist and a specific claim — it reads like analysis because it's citing a source and drawing a historical parallel. The model predicted `hype`, likely because it picked up on the emotional/dramatic framing ("KD Nets eliminated") rather than the underlying argumentative structure. With only 30 training examples for analysis, the model never learned to distinguish sourced reasoning from excited reaction posts.

**Error #2**
- **Text:** "Wembanyama is already better than any big man the NBA has seen since prime Shaq"
- **True label:** `hot_take`
- **Predicted:** `hype` (confidence: 0.21)
- **Analysis:** This is a textbook hot take — bold claim, no evidence, strong assertion. The model predicted `hype`, probably because the post celebrates a player in superlative terms, which superficially resembles a hype post. The boundary between `hot_take` and `hype` is subtle when the hot take is positive rather than controversial: both involve enthusiasm, but one asserts a claim and the other just reacts to a moment. The model never learned that distinction.

**Error #3**
- **Text:** "[Krawczynski] Timberwolves have held discussions on Giannis, Kyrie, Trey Murphy III, Josh Giddey, Derrick White"
- **True label:** `news`
- **Predicted:** `hype` (confidence: 0.21)
- **Analysis:** This has a journalist tag in brackets — a very strong signal for `news` — and lists specific names and a transaction context. The model still predicted `hype`. This suggests the model didn't learn the `[Reporter]` bracket pattern at all, which the Groq baseline handled easily. At 140 training examples spread across 5 labels, there weren't enough news examples for the model to reliably associate bracket-tagged posts with the `news` label.

### Sample Classifications

| Post | Predicted | Confidence | Correct? |
|------|-----------|------------|----------|
| "Who do you think will be the Ajay Mitchell in this year's upcoming draft?" | `question` | 0.81 | Yes |
| "14 Years Ago Today - Chris Bosh Pours Champagne After Winning Championship" | `hype` | 0.74 | Yes |
| "[Charania] BREAKING: Dusty May agreed to become new head coach of Dallas Mavericks" | `hype` | 0.21 | No (true: `news`) |
| "Trae Young will never win a championship. His defense is just too bad." | `hype` | 0.22 | No (true: `hot_take`) |
| "Steph Curry drops 52 on 9-15 from three in a blowout win" | `hype` | 0.79 | Yes |

The `question` prediction is reasonable: the post is clearly soliciting community input with no stated position, and the model likely picked up on the interrogative structure. The `hype` predictions for `news` and `hot_take` illustrate the collapse problem — the model defaulted to its strongest learned class.

### Reflection

The fine-tuned model learned two things well: `question` (clear interrogative structure) and `hype` (as a catch-all for everything else). What it failed to learn was any of the more semantically subtle distinctions — `analysis` vs `hot_take`, `news` vs `hype`, or the journalist-tag signal for `news`.

The gap between what I intended and what the model captured is significant. I intended the model to learn argumentative structure: does this post reason with evidence, or assert without it? Instead, the model learned surface vocabulary and format cues, and when those cues were ambiguous, it defaulted to `hype` — the most common pattern in excited sports discourse.

The likely reasons: 140 training examples is genuinely too small to learn 5 fine-grained distinctions, especially when 4 of the 5 labels involve opinion-heavy language that overlaps. The Groq baseline outperformed the fine-tuned model substantially (0.733 vs 0.467) because llama-3.3-70b already has world knowledge about what journalist tags mean, what a hot take looks like, and how question posts differ from assertions — things the fine-tuned DistilBERT had to learn from scratch with very little data.

---

## Spec Reflection

The spec helped most in the label design phase — the requirement to write decision rules for edge cases before annotating forced me to think carefully about the `analysis` vs `hot_take` boundary before labeling 200 examples. Without that, I would have applied those labels inconsistently across the dataset, which would have made the training signal even noisier.

Where the implementation diverged: the spec assumes fine-tuning will improve on the baseline, and the evaluation report template is structured around "fine-tuning improvement." My fine-tuned model regressed significantly. I kept the honest numbers rather than adjusting the setup to get better results, because the regression itself is informative — it reveals both the data size limitation and how much of the task's difficulty the Groq baseline handles through world knowledge rather than task-specific training.

---

## AI Usage

1. **Data collection and labeling:** An LLM was used to pre-label the 200 posts after being given the label definitions from `planning.md`. Every pre-assigned label was reviewed manually and corrected where needed. Approximately 15-20% of pre-assigned labels were changed during review, primarily on `analysis` vs `hot_take` borderline cases.

2. **README and planning.md drafting:** Claude was used to generate initial drafts of both documents given the label taxonomy, dataset stats, and notebook output. The error analysis sections, reflection, and hard case decisions were written and verified by hand using actual notebook output.

3. **Error pattern analysis:** After running the notebook, the wrong predictions were reviewed to identify the systematic `hype`-collapse pattern described in the error analysis. The pattern was confirmed by reading the confusion matrix directly.

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