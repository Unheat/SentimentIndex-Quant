# AlphaSift: Planning and Design Document

## 1. Project Goal & Community
The objective is to build a classification pipeline capable of categorizing retail investment discourse on Reddit. This targets communities like `r/stocks`, `r/wallstreetbets`, and `r/investing` to separate high-quality fundamental analysis from emotional market reactions and internet noise. 

## 2. Label Definitions
To create a distinct separation boundary, I defined the following three categories:
* **DD_RESEARCH:** Text containing explicit corporate financial numbers, ratios, percentages, growth metrics, earnings data, or deep technical infrastructure details (e.g., P/E ratio, Q3 revenue).
* **VIBE_HYPOTHESIS:** Confident directional market predictions based entirely on consumer brand feelings, anecdotes, or general sentiment without supporting hard financial metrics.
* **NOISE_MEME:** Short emotional reactions, internet investing slang, general questions completely lacking financial context, or sensationalist text.

## 3. Data Collection Plan
* **Source:** Apify Reddit Scraper targeting the `/top/?t=month` feeds across multiple subreddits.
* **Volume Target:** ~250 total rows to ensure sufficient training data while respecting Hugging Face resource limits.
* **Formatting:** Combine `title` and `body` into a unified `[TITLE] ... [BODY] ...` string to give the model explicit structural boundaries.

## 4. Annotation & Edge Cases
Due to the volume of text, manual annotation was augmented with an LLM-as-a-judge heuristic script. 
* **Edge Case Encountered:** Posts that contain a stock ticker but only ask a general question (e.g., "What do you think about MSFT?").
* **Resolution Rule:** Unless the post provides actual numbers to back up a claim, it defaults to `VIBE_HYPOTHESIS` or `NOISE_MEME`. 

## 5. Evaluation Metrics Reasoning
While overall accuracy is tracked, the primary evaluation metric is the **Macro F1-Score**. Because investment subreddits heavily skew toward low-effort noise or highly upvoted due diligence, severe class imbalance is expected. Macro F1 ensures the model's performance is evaluated equally across all classes, preventing it from "cheating" by simply guessing the majority class.

## 6. AI Tool Plan
* **Data Engineering:** Use AI to write pandas scripts to drop unused columns, strip hidden characters from CSV headers, and concatenate text features.
* **Pipeline Debugging:** Use AI to optimize string matching logic in the validation loops when dealing with unpredictable LLM API outputs.