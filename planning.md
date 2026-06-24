# TakeMeter — Project Plan

### Community Selection
I chose **r/cscareerquestions**, which is a subreddit where computer science students and professionals talk about job hunting, career advice, tech layoffs, and industry trends. This community is a perfect fit for a discourse quality classifier because the quality of the posts varies a lot. Some posts are super detailed and back up their claims with specific data points like interview counts, exact company names, and salary numbers. Others are just huge, sweeping claims about the tech industry stated with total confidence but zero evidence. Then you have posts that are just pure emotional reactions to a single event, like getting a rejection or landing an offer. Regular users can easily tell these types of posts apart, but it is hard to define them precisely. That makes it a great machine learning problem for a classifier.

### Labels and Edge Cases
I am breaking the posts down into three main categories:

* **analysis:** The post makes a structured argument backed by specific details. This includes real numbers, named companies, direct personal experience, or a clear comparison. The core claims are supported by actual evidence.
  * *Example 1:* "I interviewed at 12 companies after my layoff. Startups under 50 people ghosted within a week, but FAANG-adjacent companies took 3-4 weeks and always gave a final answer."
  * *Example 2:* "Switched from a CS degree job search to a bootcamp-style approach. I sent 80 applications with a portfolio-first resume versus 80 with a traditional one. The portfolio-first resume got 3x more callbacks."
* **hot_take:** A confident, sweeping claim about the tech industry or career paths with little to no supporting evidence. The claim might be true, but the author just asserts it rather than actually arguing it.
  * *Example 1:* "CS degrees are completely worthless now, just learn to grind LeetCode instead."
  * *Example 2:* "Nobody should bother with a master's degree, it's a scam at this point."
* **reaction:** An immediate emotional response to a specific personal event like a rejection, a layoff, or a job offer. There is basically no argument here. The post is just expressing how the user feels, not making a broader claim about the tech field.
  * *Example 1:* "Just got laid off after 2 years. I don't even know what to do right now."
  * *Example 2:* "Got my first offer ever today, I'm shaking, I can't believe it actually happened."

#### Handling Hard Edge Cases
The hardest part will be dealing with posts that start out as a personal reaction but quickly turn into a hot take. This usually looks like someone using a personal event as a one-line setup to make a massive generalization. 

* *Example:* "Got rejected again. Companies don't even care about skills anymore, they just want 5 years of experience for an entry-level role."

**Decision Rule:** I will label the post based on what dominates the actual content, not just the opening line. If the personal event is just a launchpad and the rest of the post is an unsupported generalization about the industry, I will label it a `hot_take`. If the post stays focused on the personal event and how it felt, it will be a `reaction`. The example above would be a `hot_take` because the rejection is just a one-sentence setup for a broad claim about hiring practices.

Another tricky case is when a post cites exactly one personal data point (like "I applied to 200 jobs and got 2 callbacks") and then concludes that "the market is dead." 

**Decision Rule:** If the post includes any concrete numbers or evidence tied to personal experience, I will label it as `analysis`, even if the final conclusion overreaches a bit. The evidence is still real. Pure assertions with zero personal data points will stay as `hot_take`.

### Data Collection Plan
* **Source:** Public posts and top-level comments from r/cscareerquestions. I will pull these using Reddit's public JSON endpoint, which does not require authentication for public data.
* **Volume Target:** I am aiming for around 200 total examples with a roughly even distribution. I want to make sure no single label makes up more than 70% of the dataset. 
* **Balancing the Data:** Reaction and hot take posts are usually shorter and show up more when sorting by "Hot." Analysis posts are usually longer and show up more under "Top" since they get more upvotes. I will pull from both sorting methods to keep things balanced. If a label like `analysis` is still underrepresented after the first pass, I will search for specific keywords like "interviewed at," "data point," or "in my experience" to target those posts specifically.

### Evaluation Metrics and Success
Accuracy by itself is not enough because the dataset might still end up imbalanced. If reaction posts end up being the majority, a model could get a deceptively high accuracy score just by guessing "reaction" every time. To fix this, I will track and report:

1. **Overall Accuracy:** For a quick look at how the baseline model compares to the fine-tuned one.
2. **Precision, Recall, and F1-Score (per class):** This will help catch if the model is strong on one label but completely ignoring another. For example, high precision but low recall on `analysis` would mean the model is being way too conservative when labeling evidence-backed posts.
3. **Confusion Matrix:** This will show exactly which labels are getting mixed up with each other, which is super helpful for troubleshooting.

#### Definition of Success
For this project to be a success, the classifier needs to hit an F1-score of at least 0.70 across all three classes. Also, the fine-tuned model needs to beat the zero-shot baseline by at least 10 points in accuracy, otherwise the fine-tuning process was not really worth it. For an actual real-world deployment in a moderation tool, I would want all classes above a 0.75 F1-score so that it does not silently fail on an entire category.

### AI Tool Usage Plan
* **Label Stress-Testing:** Before I spend time annotating all 200 rows, I will give my label definitions and edge case rules to an AI tool. I will ask it to generate 8 to 10 borderline, realistic r/cscareerquestions posts. If I struggle to manually label the AI-generated posts, I know I need to tighten up my definitions before starting.
* **Annotation Assistance:** I plan to use an LLM (Groq's llama-3.3-70b-versatile, which is also my baseline model) to pre-label the scraped posts. Then, I will manually review and correct every single label myself before finalizing the data. I will keep track of which rows were pre-labeled by using a temporary `pre_labeled` column in my spreadsheet, and I will disclose this in the project's README under the AI usage section.
* **Failure Analysis:** After fine-tuning, I will feed the model's incorrect predictions back into an AI tool and ask it to find common patterns, like if it struggles with short posts, sarcasm, or specific label pairs. I will manually double-check these flagged examples to make sure the AI's pattern ideas are actually accurate before putting them in my final report.
