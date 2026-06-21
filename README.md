# TakeMeter: Discourse Quality Classifier for Indie Music Communities

A fine-tuned text classifier that evaluates the type of discourse in posts from r/indieheads and r/LetsTalkMusic. Given a post, TakeMeter predicts whether it is structured **analysis**, an unsupported **hot_take**, or an immediate **reaction**.

---

## Community Choice

I chose r/indieheads and r/LetsTalkMusic because these subreddits are text-heavy and produce a wide range of discourse quality. Users regularly post album reviews with production-level specifics, bold opinions with no backing, and emotional reactions to new releases — often in the same thread. This variation makes the community a natural fit for a discourse classification task. The distinctions I'm measuring reflect how people in these communities actually evaluate each other's contributions: "that's just a hot take" vs. "this is actually a great analysis" are phrases that appear in the wild here.

---

## Label Taxonomy

### `analysis`
A post that makes a structured argument about music backed by specific evidence — production details, historical comparisons, chart data, lyrical breakdowns, or genre context. The claim is supported, not just asserted.

- **Example 1:** "The reason Phoebe Bridgers works so well is that her production choices always mirror the emotional content of the lyric — sparse arrangements for grief, fuller textures when the narrator is performing okayness rather than feeling it."
- **Example 2:** "Sufjan Stevens uses orchestration as emotional punctuation on Illinois. The strings don't add texture — they mark the moments where the narrator can't speak anymore."

### `hot_take`
A bold, confident opinion stated without supporting evidence. The post asserts a claim rather than argues for it. The opinion might be true, but the post doesn't back it up with specifics.

- **Example 1:** "Phoebe Bridgers is the most overrated artist of the last decade. People only like her because she seems sad."
- **Example 2:** "Tame Impala is just dad rock for millennials who think they have good taste."

### `reaction`
An immediate emotional response to a specific release, concert, or music news event. Little to no argument — the post is expressing a feeling in the moment rather than making a claim.

- **Example 1:** "just heard For Emma Forever Ago for the first time and i am genuinely not okay. that album destroyed me."
- **Example 2:** "ok i finally listened to In the Aeroplane Over the Sea and i understand now. i get it. i'm crying."

---

## Dataset

**Source:** Public posts and top-level comments from r/indieheads and r/LetsTalkMusic on Reddit, collected manually.

**Size:** 200 labeled examples

**Label distribution:**

| Label | Count |
|-------|-------|
| analysis | 71 |
| hot_take | 67 |
| reaction | 62 |
| **Total** | **200** |

**Labeling process:** Each post was read in full before labeling. Labels were assigned using the definitions above. Posts that fell on a boundary (e.g., a one-stat hot take) were resolved using the decision rules documented in planning.md.

**Split:** 70% train / 15% validation / 15% test (stratified by label, random_state=42)

### Difficult-to-label examples

**1. The one-stat hot take**
> "Radiohead hasn't made a good album since Kid A — everything after has declining Metacritic scores."

This looks like analysis because it cites a specific metric, but the stat is cherry-picked and decorative — removing it leaves a fully intact hot take. Decision rule: if the evidence wouldn't hold up as an actual argument without the opinion framing, it's `hot_take`. → **Labeled: hot_take**

**2. The enthusiastic analysis**
> "I'm OBSESSED with how the production on this track layers the drums — the way they drop out in the bridge completely changes the emotional weight of the vocal."

The excited tone reads like a reaction, but the post contains a specific, verifiable musical observation about a production choice and its effect. Decision rule: if the post contains specific musical observations, label it `analysis` even if the tone is excited. → **Labeled: analysis**

**3. The long reaction**
> Several posts ran 3–4 sentences describing a listening experience in detail without making any structured argument — just an extended expression of feeling. Decision rule: length doesn't change the label. If the post is expressing feelings about a release without making a structured argument, it's `reaction` regardless of length. → **Labeled: reaction**

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace)

**Training setup:** Fine-tuned on Google Colab with a T4 GPU. Training library: HuggingFace `transformers` + `datasets`.

**Hyperparameters:**
- Epochs: 3
- Learning rate: 2e-5
- Batch size: 16
- Max token length: 256

**Hyperparameter decision:** The default learning rate of 2e-5 was kept after observing stable validation loss across epochs. A higher rate (5e-5) caused the validation loss to spike on epoch 2, suggesting overfitting on the small dataset, so the default was retained.

---

## Baseline

**Model:** `llama-3.3-70b-versatile` via Groq API (zero-shot, no task-specific training)

**Prompt approach:** The system prompt included the community context, all three label definitions as written in planning.md, one example post per label, and an instruction to output only the label name. Temperature was set to 0 for deterministic output.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|-------|----------|
| Zero-shot baseline (Llama 3.3 70B) | 1.000 |
| Fine-tuned DistilBERT | 0.700 |

The baseline achieved 100% on the 30-example test set — the task is within zero-shot capabilities of a large model given well-defined labels. Fine-tuning a small model (DistilBERT) on 200 examples performed worse than prompting a large one, which is itself a meaningful finding. Note: 3 examples overlapped between train and test sets due to a data collection issue; these are disclosed but do not explain the baseline result.

### Per-Class Metrics

**Baseline (Llama 3.3 70B, zero-shot):**

| Label | Precision | Recall | F1 |
|-------|-----------|--------|----|
| analysis | 1.00 | 1.00 | 1.00 |
| hot_take | 1.00 | 1.00 | 1.00 |
| reaction | 1.00 | 1.00 | 1.00 |

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 |
|-------|-----------|--------|----|
| analysis | 0.58 | 1.00 | 0.73 |
| hot_take | 1.00 | 0.10 | 0.18 |
| reaction | 1.00 | 1.00 | 1.00 |

### Confusion Matrix (Fine-tuned Model)

|  | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|--|---------------------|---------------------|---------------------|
| **True: analysis** | 11 | 0 | 0 |
| **True: hot_take** | 8 | 1 | 1 |
| **True: reaction** | 0 | 0 | 9 |

### Wrong Predictions Analysis

All 9 wrong predictions were `hot_take` examples misclassified as `analysis` (8 cases) or `reaction` (1 case). No other label pair was confused.

**Example 1:**
- Post: "Low's last three albums are the best run of late career albums in indie history."
- True label: hot_take
- Predicted: analysis (confidence: 0.37)
- Analysis: The post references a specific discography ("last three albums") which the model reads as evidence. But the claim is pure assertion — no production details, comparisons, or argument. The model keyed on artist/discography specificity rather than argument structure.

**Example 2:**
- Post: "Neutral Milk Hotel fans are more in love with the mythology than the actual music."
- True label: hot_take
- Predicted: analysis (confidence: 0.37)
- Analysis: This post makes a sociological observation about a fanbase, which has the surface structure of an argument. But it provides no evidence — no examples, no comparisons, no specifics. The model couldn't distinguish "sounds like a claim" from "is an argument."

**Example 3:**
- Post: "Phoebe Bridgers is the most overrated artist of the last decade. People only like her because she seems sad."
- True label: hot_take
- Predicted: analysis (confidence: 0.34)
- Analysis: This exact post appears as a `hot_take` example in the README. The model still misclassifies it, suggesting `hot_take` signal in the training data is too weak — the model defaulted to `analysis` even on a textbook case.

### Sample Classifications

| Post (truncated) | Predicted Label | Confidence |
|------------------|-----------------|------------|
| "just heard For Emma Forever Ago for the first time and i am genuinely not okay..." | reaction | ~0.99 |
| "the reason OK Computer holds up thirty years later is structural: Radiohead sequenced it so..." | analysis | ~0.91 |
| "Tame Impala is just dad rock for millennials who think they have good taste." | analysis | ~0.36 |
| "Snail Mail is a better live performer than studio artist and the recordings don't capture what she does in person." | analysis | ~0.39 |
| "listened to Fetch the Bolt Cutters all the way through for the first time and i need to lie down for a week." | reaction | ~0.98 |

The first two predictions are correct and confident. The third and fourth are wrong and low-confidence — the model knows it's uncertain on `hot_take` examples but defaults to `analysis` anyway.

--- 

## Reflection: What the Model Learned vs. What I Intended

The model learned two things well: `reaction` (emotional, informal language, first-person present tense) and `analysis` (specific artist/album references, technical vocabulary, structured sentences). It learned almost nothing about `hot_take`.

The intended boundary between `hot_take` and `analysis` was argument structure — does the post support its claim with evidence? The model instead learned surface features: posts with named artists and discography references look like `analysis` regardless of whether they argue anything. A hot take about a specific album ("Low's last three albums are the best run in indie history") has the same surface features as an analysis post, so the model collapses them.

This is partly a data problem. With only ~67 `hot_take` training examples and significant surface overlap with `analysis`, the model didn't see enough signal to learn the structural distinction. It would likely require either more training data, explicit structural features (e.g., presence of "because," "since," "which means"), or a larger base model to capture the difference between assertion and argument.

--- 

## Spec Reflection

**One way the spec helped:** The requirement to run a zero-shot baseline before fine-tuning forced an honest comparison. Without that constraint I would have reported 70% accuracy as a standalone result — having the baseline revealed that a large zero-shot model outperforms fine-tuned DistilBERT on this task, which is the most interesting finding in the project.

**One way implementation diverged:** The spec assumes fine-tuning will outperform the baseline. My results went the other direction — the baseline scored 100% and fine-tuning scored 70%. Rather than treating this as a failure, I documented it honestly as a finding about task difficulty and model scale tradeoffs.

---

## AI Usage

**1. Label stress-testing:** I gave Claude my three label definitions and asked it to generate posts that sit at the boundary between `analysis` and `hot_take`. Several generated examples used one specific statistic in an otherwise unsupported opinion — this stress-test directly led to the "one-stat hot take" decision rule documented in the Hard Edge Cases section.

**2. Annotation assistance:** I used Claude to pre-label batches of 20 posts at a time by providing my label definitions and raw post text. Every pre-assigned label was reviewed and corrected before being added to the CSV. Examples that were pre-labeled are noted in the `notes` column of the dataset.

**3. Failure analysis:** After fine-tuning, I pasted all 9 misclassified examples into Claude and asked it to identify common patterns. It identified that all errors involved `hot_take` posts with named artists or discography references, which the model was reading as evidence of analysis. I verified this pattern by re-reading every misclassified example myself before writing the analysis in the evaluation report.