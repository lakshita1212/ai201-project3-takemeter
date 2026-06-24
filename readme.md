# TakeMeter — Discourse Quality Classifier for Hacker News Career Discussions

A fine-tuned text classifier that evaluates discourse quality in Hacker News
comments about tech careers, layoffs, salaries, hiring, and the job market.

## Community

I chose **Hacker News (news.ycombinator.com)**, specifically comments on
stories and "Ask HN" threads about tech careers, layoffs, salaries, hiring,
and the job market. This is a strong fit for a discourse-quality classifier
because comment quality varies enormously: some comments are detailed
personal accounts backed by specifics (real numbers, named companies,
concrete personal experience), some are sweeping industry-wide claims stated
with total confidence and little evidence, and some are pure emotional
responses to a personal event (a layoff, a rejection, a frustrating
interview experience). These three modes are easy for a regular HN reader to
recognize intuitively but hard to define precisely — exactly the kind of
subjective-but-learnable distinction this project asks for.

I originally planned to collect data from r/cscareerquestions on Reddit, but
Reddit's API blocks requests from cloud environments like Google Colab
(403 errors) and its developer-app registration page was inaccessible due to
a separate network issue. I switched to Hacker News, which covers the same
discourse domain — tech career discussion — and has a fully open,
unauthenticated public API.

## Label Taxonomy

- **analysis** — The comment makes a structured argument backed by
  specifics: real numbers, named companies, direct personal experience, or a
  clear comparison. The claim is supported by evidence that would hold up
  even if the opinion framing were removed.
  - Example: "I interviewed at 12 companies after my layoff. Startups under
    50 people ghosted within a week, but FAANG-adjacent companies took 3-4
    weeks and always gave a final answer."
  - Example: "I work at US based company and I earn $900k. Remote work has
    been a boon to me."

- **hot_take** — A confident, sweeping claim about the industry or career
  path, stated with little or no supporting evidence. The claim might be
  true, but the comment asserts rather than argues.
  - Example: "CS degrees are completely worthless now, just learn to grind
    LeetCode instead."
  - Example: "Stop thinking they want to hire the best talent. FAANG isn't
    trying to hire the best talent. They're looking for people who will be
    productive at FAANG."

- **reaction** — An immediate emotional response to a specific personal
  event (a rejection, a layoff, an offer, an interview). Little to no
  argument — the comment expresses how something feels rather than making a
  claim about the field.
  - Example: "Hundreds of applications since I was laid off in November.
    Working a trash entry level job in an unrelated field to try and pay
    the bills. The market is absolute garbage."
  - Example: "A no-reply@ email address? This seriously happened? You
    dodged a bullet."

## Data Collection, Labeling Process, and Label Distribution

**Source:** Comments from Hacker News stories, collected via two free public
APIs: HN's Algolia search API (to find relevant career/tech-discourse
stories using terms like "career," "layoffs," "salary," "job market,"
"hiring," and "interview") and HN's Firebase API (to pull comment text from
each story's comment tree). Neither requires authentication.

**Cleaning:** The initial scrape returned 1,507 comments. After random
sampling down to a working set, I discovered the scrape had picked up a
meaningful number of job postings from "Who's Hiring"-style threads (e.g.,
"Quora is hiring in Palo Alto, CA...") mixed in with genuine discourse.
These aren't discourse at all — there's no claim or argument to classify —
so I manually identified and removed 39 job-posting rows across several
review passes.

**Labeling process:** I used Groq's `llama-3.3-70b-versatile` to pre-label
an initial batch of 300 comments using my label definitions (disclosed in
the AI Usage section below). Reviewing the pre-labeled output, I found Groq
substantially over-predicted `analysis` — many comments that were really
generic advice or sweeping opinions with no real evidence were being
labeled `analysis` rather than `hot_take`. I manually re-read and
re-labeled every comment in the final dataset myself against the
definitions above, correcting these errors and assigning fresh labels to
comments Groq couldn't classify (some calls failed due to hitting Groq's
free-tier daily rate limit).

**Final dataset:** 221 labeled comments.

| Label | Count | Percentage |
|---|---|---|
| analysis | 120 | 54.3% |
| hot_take | 80 | 36.2% |
| reaction | 21 | 9.5% |

No label exceeds 70% of the dataset, but `reaction` is notably thin at
9.5% — a limitation discussed further in the Reflection section below.

### Three Difficult Labeling Decisions

1. **"Yes, it's really that bad. Even trying to change teams internally at
   my company is the hardest it's ever been in over a decade of
   experience."** — This cites a real personal fact (a decade of tenure)
   but makes no structured argument with it; it's a felt assessment, not
   evidence-backed reasoning. I labeled it `reaction` because the function
   of the comment is emotional confirmation, not argument, despite the
   passing mention of experience.

2. **"Just by curiosity, is there any particular reason why the math and
   physics teacher job does not make you happy?"** — This is a sincere
   question, which doesn't fit cleanly into any of the three labels (it's
   not an argument, and it's not really an emotional response either). I
   labeled it `reaction` by elimination — it has no argument structure, and
   its personally-invested, curious tone sits closer to the emotional/
   personal register than an assertion does. This exposed a real gap in my
   taxonomy: it doesn't have a clean category for sincere questions.

3. **"I created an account just so that I could comment. I'm not a dev
   although perhaps in a few years hope to pivot..."** — The opening
   framing signals strong personal/emotional motivation (breaking lurker
   silence to weigh in), but the comment goes on to offer actual advice. I
   labeled it `reaction` because the framing dominates the comment's
   character, even though structured suggestions follow.

## Fine-Tuning Approach

- **Base model:** `distilbert-base-uncased`, fine-tuned with a 3-class
  classification head.
- **Training setup:** 70% / 15% / 15% train/validation/test split
  (stratified by label), 3 training epochs, learning rate 2e-5, batch size
  16 — the notebook's default hyperparameters, chosen because they are the
  standard starting point for fine-tuning BERT-family models on small
  datasets and the project's guidance notes these work well for datasets in
  the 100–500 example range.
- **Key hyperparameter decision:** I kept the default 3 epochs rather than
  increasing it. With only ~154 training examples (70% of 221) split across
  3 classes — and `reaction` having only ~15 training examples after the
  split — more epochs risked overfitting on an already-small dataset rather
  than improving generalization.

## Baseline Description

The baseline used Groq's `llama-3.3-70b-versatile` in a zero-shot setting:
the same three label definitions and one example per label (as written
above) were provided in a system prompt, and the model was instructed to
respond with only the label name. The baseline was run once on the locked
test set (34 examples) with `temperature=0` for deterministic output. All
34 responses were successfully parsed into one of the three valid labels.

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq llama-3.3-70b-versatile) | 0.765 |
| Fine-tuned DistilBERT | 0.559 |

The fine-tuned model performed **20.6 points worse** than the zero-shot
baseline — a regression, not an improvement.

### Per-Class Metrics

**Zero-shot baseline (Groq):**

| Label | Precision | Recall | F1 |
|---|---|---|---|
| analysis | 0.78 | 0.95 | 0.86 |
| hot_take | 0.83 | 0.42 | 0.56 |
| reaction | 0.60 | 1.00 | 0.75 |

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 |
|---|---|---|---|
| analysis | 0.56 | 1.00 | 0.72 |
| hot_take | 0.00 | 0.00 | 0.00 |
| reaction | 0.00 | 0.00 | 0.00 |

### Confusion Matrix (Fine-Tuned Model, Test Set)

| True \ Predicted | analysis | hot_take | reaction |
|---|---|---|---|
| **analysis** | 19 | 0 | 0 |
| **hot_take** | 12 | 0 | 0 |
| **reaction** | 3 | 0 | 0 |

*(See `confusion_matrix.png` in the repo for the rendered version.)*

The fine-tuned model predicted `analysis` for every single test example,
regardless of true label. This is a complete classifier collapse onto the
majority class, not a model that learned partial distinctions.

### Three Wrong Predictions, Analyzed

1. **"Although it seems like there are endless start ups with 22 year old
   CEOs and executive, it's still easier to do business as a C level
   executive if you're a bit older..."**
   True: `hot_take` → Predicted: `analysis` (confidence: 0.37)
   *Why it failed:* This is a sweeping generalization about executive age
   with no supporting evidence — a textbook `hot_take` by my definition.
   The model's confidence (0.37) is barely above the 0.33 baseline for
   random guessing among 3 classes, meaning the model wasn't confidently
   wrong — it had essentially no real signal to distinguish this from
   `analysis`.

2. **"Related: https://news.ycombinator.com/item?id=33677860. There are
   legal versions of that approach."**
   True: `hot_take` → Predicted: `analysis` (confidence: 0.37)
   *Why it failed:* This is a very short comment with almost no content to
   classify from. The presence of a link likely confused the model into
   associating it with `analysis` (since many true `analysis` examples in
   training also contained links to evidence), even though this comment
   makes no argument at all — it's a bare assertion with a pointer
   elsewhere.

3. **"How worried should an entry level developer who has been working
   only 6 months at his current company be? Due to an extremely high cost
   of living, paying all my bills and student loans..."**
   True: `reaction` → Predicted: `analysis` (confidence: 0.38)
   *Why it failed:* This comment mixes a personal, anxious situation (cost
   of living, student loans, job security worry) with a question format.
   The model likely picked up on the specific personal details (loans,
   cost of living, tenure) as `analysis`-like signals (since real `analysis`
   examples often cite specific personal numbers), without learning that
   the *function* of this comment is anxious self-disclosure, not argument.

**Pattern:** All three wrong predictions cluster at nearly identical low
confidence (~0.37–0.38), and all three were predicted as `analysis`
regardless of true label. This points to a model that did not learn
meaningful decision boundaries between classes at all, rather than a model
that learned imperfect-but-real boundaries.

### Sample Classifications

| Comment (truncated) | Predicted Label | Confidence |
|---|---|---|
| "I turned on 'casually looking for work' on LinkedIn just because I was curio..." | analysis | 0.37 |
| "It's possible this is including many misclassified listings (e.g. hourly paid internships). Addition..." | analysis | 0.38 |
| "$200k is a lot for most positions in the US. The median for a US dev is about $105k. I'm not surpris..." | analysis | 0.37 |
| "Part of the H-1B process is to prove you (hiring company) can't find an equal American citizen to c..." | analysis | 0.38 |
| "Gergely Orosz's theory[1]: > An IRS tax code change in Section 174. This change eliminates the abil..." | analysis | 0.37 |

All five correctly-classified examples are `analysis` — consistent with the
confusion matrix showing the model predicted `analysis` for every test example.
Confidence scores cluster tightly between 0.37 and 0.38, just above the 0.33
random-guess baseline for 3 classes, which confirms the model is not making
confident, well-learned predictions even on the examples it gets right. The
correct predictions here reflect the majority-class collapse described above,
not genuine learned signal across all three categories.

## Reflection: What the Model Learned vs. What I Intended

I intended for the model to learn three distinct functional categories of
discourse: evidence-backed argument, unsupported assertion, and emotional
expression. What it actually learned was closer to a single rule: "predict
analysis." The recall for `analysis` was a perfect 1.00, while `hot_take`
and `reaction` both scored 0.00 across precision and recall — the model
never once predicted either minority class on the test set.

The most likely cause is dataset size and imbalance interacting badly: with
only 221 total examples split 70/15/15, the training set had roughly 154
examples, of which only about 36 epochs' worth of signal-equivalent rows
were available for `reaction` (which makes up just 9.5% of the full
dataset). DistilBERT, with no prior exposure to this specific task, likely
didn't see enough `hot_take` and `reaction` examples during training to
learn boundaries reliable enough to ever activate those output classes,
and instead converged on the easiest, lowest-error strategy: always
predicting the majority class.

This is reinforced by the baseline comparison: Groq's `llama-3.3-70b-
versatile`, with **zero** task-specific training, substantially outperformed
the fine-tuned model (0.765 vs. 0.559 accuracy) and successfully predicted
all three classes with reasonable recall (0.95, 0.42, and 1.00
respectively). This strongly suggests the task itself is learnable — a
sufficiently large/capable model can distinguish these three categories —
but my fine-tuning dataset was too small and too imbalanced for DistilBERT
to learn the same distinctions from scratch in 3 epochs.

If I were to improve this, the first thing I'd do is collect substantially
more `reaction` and `hot_take` examples specifically (targeted HN search
queries aimed at those discourse types) before re-running fine-tuning,
rather than adjusting hyperparameters — the problem looks like a data
volume/balance issue, not a training-configuration issue.

## Spec Reflection

The project spec's emphasis on writing `planning.md` *before* collecting
data was genuinely helpful — having concrete label definitions and example
posts written down before annotation gave me a stable reference to label
against, rather than drifting in my own judgment across 221 rows.

Where my implementation diverged from the original plan: I had planned to
collect data from r/cscareerquestions on Reddit, but switched to Hacker
News mid-project after hitting Reddit API access blocks that weren't
resolvable in a reasonable time. I also discovered partway through
annotation that my scraped dataset contained a meaningful number of job
postings that needed to be manually identified and removed — something the
original plan didn't anticipate, since "Ask HN" career threads turned out
to contain more recruiting-adjacent noise than expected.

## AI Usage

1. **Annotation assistance (pre-labeling):** I used Groq's
   `llama-3.3-70b-versatile` to pre-label an initial batch of 300 scraped HN
   comments, giving it my label definitions and one example per label (the
   same prompt used for the final baseline). I directed it to assign exactly
   one label per comment with no explanation. Reviewing the output, I found
   it substantially over-predicted `analysis` — many generic-advice or
   sweeping-opinion comments that should have been `hot_take` were labeled
   `analysis`. I manually re-reviewed and re-labeled every single comment in
   the final 221-row dataset against my definitions myself, overriding the
   AI's label wherever it didn't match my own reading of the comment.

2. **Failure pattern analysis:** After the fine-tuned model's evaluation
   run, I used an AI tool to help identify the pattern across the model's
   wrong predictions. It surfaced that all wrong predictions were predicted
   as `analysis` regardless of true label, and that confidence scores
   clustered tightly around 0.37–0.38 (just above the 0.33 random-guess
   baseline for 3 classes) — which I then verified myself by checking the
   full confusion matrix and confirming `hot_take` and `reaction` both had
   zero true positives. I used this to write the "what the model learned
   vs. intended" reflection above, but the causal explanation (dataset
   size/imbalance) is my own interpretation, which I cross-checked against
   the baseline's much stronger per-class performance on the same
   imbalanced label distribution.
