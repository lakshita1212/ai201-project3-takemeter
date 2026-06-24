# TakeMeter — Planning

## Community

I chose **Hacker News (news.ycombinator.com)**, specifically comments on "Ask HN" threads and stories related to tech careers, layoffs, salaries, hiring, and the job market. This community is a strong fit for a discourse-quality classifier for the same reason r/cscareerquestions would be: post quality varies enormously. Some comments are detailed personal accounts backed by specifics (interview counts, company names, salary numbers, named technologies), some are sweeping industry-wide claims stated with total confidence and no evidence, and some are pure emotional reactions to a single event (a rejection, a layoff, an offer). HN's career/industry discourse covers the same ground as r/cscareerquestions — the audience and topics overlap heavily — while being far easier to collect from programmatically, since HN's API is fully public with no authentication or rate-limit blocking.

## Labels

- **analysis** — The post makes a structured argument backed by specifics: real numbers, named companies, direct personal experience, or a clear comparison. The claim is supported by evidence that would hold up even if the opinion framing were removed.
  - Example 1: "I interviewed at 12 companies after my layoff. Startups under 50 people ghosted within a week, but FAANG-adjacent companies took 3-4 weeks and always gave a final answer."
  - Example 2: "Switched from a CS degree job search to a bootcamp-adjacent approach — sent 80 applications with a portfolio-first resume vs. 80 with a traditional one. Portfolio-first got 3x more callbacks."

- **hot_take** — A confident, sweeping claim about the industry or career path, stated with little or no supporting evidence. The claim might be true, but the post asserts rather than argues.
  - Example 1: "CS degrees are completely worthless now, just learn to grind LeetCode instead."
  - Example 2: "Nobody should bother with a master's degree, it's a scam at this point."

- **reaction** — An immediate emotional response to a specific personal event (a rejection, a layoff, an offer, a interview). Little to no argument — the post is expressing how something feels, not making a claim about the field.
  - Example 1: "Just got laid off after 2 years. I don't even know what to do right now."
  - Example 2: "Got my first offer ever today, I'm shaking, I can't believe it actually happened."

## Hard Edge Cases

The hardest case is a post that **starts as a reaction but pivots into a hot take**: a personal event used as a one-line setup for a sweeping claim.

Example: "Got rejected again. Companies don't even care about skills anymore, they just want 5 years of experience for an entry-level role."

**Decision rule:** Label by what dominates the post's content, not its opening line. If the personal event is just a springboard and the bulk of the post is an unsupported generalization about the industry, label it `hot_take`. If the post stays focused on the personal event and how it felt, with no broader claim, label it `reaction`. The example above is `hot_take` — the rejection is a one-sentence setup; the real content is the generalized claim about hiring practices.

A second recurring ambiguous case: a post that cites one personal data point (e.g., "I applied to 200 jobs and got 2 callbacks") and then generalizes from it ("so clearly the market is dead"). Decision rule: if the post has *any* concrete number/evidence tied to personal experience, even if the conclusion is a broad generalization, label it `analysis` — the evidence is real, even if the conclusion overreaches. Pure assertion with zero personal data point stays `hot_take`.

## Data Collection Plan

- **Source:** Comments from Hacker News stories, collected via two free public APIs: HN's Algolia search API (to find relevant career/tech-discourse stories using search terms like "career," "layoffs," "salary," "job market," "hiring," and "interview") and HN's official Firebase API (to pull the actual comment text from each story's comment tree). Neither API requires authentication.
- **Volume target:** ~200+ examples total, aiming for roughly even distribution — no label above 70% of the dataset. Since search terms were chosen to span different discourse types (e.g., "layoffs" tends to surface reaction-heavy comments, "career"/"salary" tend to surface analysis-heavy comments with real numbers), I expect reasonable natural balance across labels, but I reviewed the distribution after collection and would pull additional targeted searches if any label was underrepresented.
- **If a label is underrepresented after the first pass:** run additional Algolia searches with terms more likely to surface that label specifically — e.g., if `analysis` is underrepresented, search terms like "compensation negotiation" or "interview process" which tend to correlate with detailed, evidence-backed comments.

**Note:** I originally planned to collect from r/cscareerquestions on Reddit, but Reddit's `.json` endpoint returns 403 errors when accessed from cloud environments like Google Colab (Reddit blocks datacenter IP ranges), and Reddit's developer app-registration page was also inaccessible due to a network/browser issue. I switched to Hacker News, which covers the same tech-career discourse domain and has a fully open, unauthenticated API that doesn't block cloud IPs.

## Evaluation Metrics

Accuracy alone isn't enough because the dataset may have label imbalance (e.g., if `reaction` posts are more common), in which case a model could get a deceptively high accuracy by mostly predicting the majority class. I'll report:
- **Overall accuracy** — for a quick top-line comparison between baseline and fine-tuned model.
- **Per-class precision, recall, and F1** — to catch cases where the model is strong on one label but ignores another (e.g., high precision but low recall on `analysis` would mean the model is too conservative in calling out evidence-backed posts).
- **Confusion matrix** — to see *which* labels get confused with which, since the direction of the error (e.g., `analysis` predicted as `hot_take`) is diagnostically useful, not just the fact that an error occurred.

## Definition of Success

A genuinely useful classifier should hit **at least 0.70 F1 on every class**, and the fine-tuned model should **meaningfully outperform the zero-shot baseline** (not just by 1-2 points — at least 10 points of accuracy improvement, since otherwise fine-tuning added nothing). For "good enough for real deployment" in a community moderation/tagging tool, I'd want all three classes above 0.75 F1 with no single class collapsing to near-zero, since a model that silently fails on one entire category (e.g., never correctly catching `analysis`) would be misleading to end users even if overall accuracy looked fine.

## AI Tool Plan

- **Label stress-testing:** Before annotating the full 200, I'll give an AI tool my three label definitions and the edge case rules above, and ask it to generate 8-10 borderline r/cscareerquestions-style posts. If I can't cleanly assign labels to several of them, I'll tighten the definitions before committing to full annotation.
- **Annotation assistance:** I plan to use an LLM (Groq's llama-3.3-70b-versatile, same as the baseline model) to pre-label a batch of scraped posts using my label definitions, then manually review and correct every pre-assigned label before finalizing the CSV. I will track which examples were pre-labeled by keeping a `pre_labeled` column in my working spreadsheet (removed before final CSV submission) for disclosure purposes in the README's AI usage section.
- **Failure analysis:** After fine-tuning, I'll paste the model's list of wrong predictions into an AI tool and ask it to identify common themes (e.g., short posts, sarcasm, a specific label pair that's consistently confused). I'll then re-read the flagged examples myself to verify the pattern actually holds before including it in the evaluation report — the AI's pattern suggestion is a starting point, not the final word.
