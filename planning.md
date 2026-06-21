# TakeMeter — planning.md

> Written before data collection and implementation.

---

## Community

I chose the music community, specifically r/indieheads and r/LetsTalkMusic on Reddit. These subreddits are text-heavy and discourse-rich — users regularly post album reviews, artist opinions, reaction posts to new releases, and structured arguments about music theory or cultural impact. The discourse varies enormously in quality: some posts cite production techniques and historical comparisons while others are purely emotional reactions. This variation makes it an ideal community for a discourse quality classifier. The distinctions I'm measuring reflect how people in these communities actually evaluate each other's posts — "that's just a hot take" vs. "this is actually a great analysis."

---

## Labels

### analysis
A post that makes a structured argument about music backed by specific evidence — production details, historical comparisons, chart data, lyrical breakdowns, or genre context. The claim is supported, not just asserted.

**Example 1:** "Mitski's 'Laurel Hell' deliberately borrows 80s synth-pop production to contrast the cold, mechanical sound with emotionally raw lyrics — this tension is what makes the album work where her earlier records relied purely on folk intimacy."

**Example 2:** "The reason Kendrick's TPAB holds up better than DAMN is structural: TPAB has a narrative arc across all 16 tracks whereas DAMN is a collection of strong singles without thematic cohesion. You can hear this in how the interludes function differently on each record."

### hot_take
A bold, confident opinion stated without supporting evidence. The post asserts a claim rather than argues for it. The opinion might be true, but the post doesn't back it up with specifics.

**Example 1:** "Taylor Swift has never made a genuinely good album. Fearless was overrated then and it's overrated now."

**Example 2:** "Frank Ocean is the most overrated artist of the last decade. People only like him because he's mysterious, not because the music is actually good."

### reaction
An immediate emotional response to a specific release, concert, or music news event. Little to no argument — the post is expressing a feeling in the moment rather than making a claim.

**Example 1:** "Just heard the new Boygenius track and I'm actually crying. This album is going to destroy me."

**Example 2:** "Charli xcx dropped the album and I haven't moved from my couch in 3 hours. Send help."

---

## Hard Edge Cases

**The one-stat hot take:** A post that cites one specific fact but uses it to make an accusatory or exaggerated claim. Example: "Radiohead hasn't made a good album since Kid A — everything after has declining Metacritic scores." This looks like analysis (cites scores) but the stat is cherry-picked and decorative rather than part of a structured argument. Decision rule: if removing the statistic would leave a fully intact hot take, label it hot_take.

**The enthusiastic analysis:** A post that uses emotional language but contains a real structured argument underneath. Example: "I'm OBSESSED with how the production on this track layers the drums — the way they drop out in the bridge completely changes the emotional weight." Decision rule: if the post contains specific, verifiable musical observations, label it analysis even if the tone is excited.

**The long reaction:** A post that is clearly a reaction but runs several paragraphs. Decision rule: length doesn't change the label. If the post is expressing feelings about a specific release without making a structured argument, it's reaction regardless of how long it is.

---

## Data Collection Plan

**Source:** r/indieheads and r/LetsTalkMusic on Reddit. Public posts and top-level comments only.

**Target:** 200+ examples with roughly equal distribution across 3 labels (~67 per label). Will aim for 70 per label to have buffer.

**Process:** Manually copy posts and comments into a CSV file with columns: text, label, notes. Will read each post fully before labeling.

**If a label is underrepresented:** If any label falls below 55 examples after initial collection, collect more targeted examples — for reactions, look at new release threads; for analysis, look at "deep dive" or "appreciation" posts; for hot takes, look at controversial opinion threads.

---

## Evaluation Metrics

**Overall accuracy:** Baseline comparison metric — tells us if fine-tuning helped at all.

**Per-class F1:** The most important metric for this task. Because all three classes matter equally and the dataset is roughly balanced, F1 per class tells us which boundaries the model learned and which it didn't.

**Confusion matrix:** Shows directional errors — which label pairs get confused and in which direction. Essential for diagnosing whether the model is conflating hot_take with analysis or reaction with hot_take.

Accuracy alone is insufficient because a model that predicts "hot_take" for everything could achieve 33% accuracy while learning nothing useful.

---

## Definition of Success

A fine-tuned model that achieves at least 70% overall accuracy and per-class F1 above 0.65 for all three labels would be genuinely useful as a community moderation or recommendation tool. The baseline (zero-shot LLM) will likely achieve 50-65% — fine-tuning should meaningfully exceed this. If per-class F1 for any label falls below 0.50, that label's boundary needs redesign.

---

## AI Tool Plan

**Label stress-testing:** Give Claude my three label definitions and ask it to generate 10 posts that sit at the boundary between analysis and hot_take (the hardest pair). If I can't cleanly label them, tighten the definitions before annotating 200 examples.

**Annotation assistance:** Use Claude to pre-label batches of 20 posts at a time by giving it my label definitions and the raw text. I will review and correct every pre-assigned label before adding it to the CSV. I will track which examples were pre-labeled in the notes column.

**Failure analysis:** After fine-tuning, paste all misclassified examples into Claude and ask it to identify common themes — post length, sarcasm, missing context, label pair confusion. I will verify every pattern it identifies by re-reading the examples myself before including it in the evaluation report.