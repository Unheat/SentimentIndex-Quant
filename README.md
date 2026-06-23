# AlphaSift: Quantitative NLP Classifier for Retail Financial Discourse

## Overview
AlphaSift is an end-to-end Natural Language Processing pipeline designed to classify retail investment discourse on Reddit into actionable categories. The project ingests raw social media data, pre-processes the text, and fine-tunes a `distilbert-base-uncased` transformer model to differentiate between hard financial research, sentiment-based predictions, and general internet noise.

## Evaluation Results

### Performance Comparison
| Model | Overall Accuracy | Macro F1-Score |
| :--- | :--- | :--- |
| **Zero-Shot Baseline (Llama-3.3-70b)** | 61.5% | 0.45 |
| **Fine-Tuned Classifier (DistilBERT)** | 59.0% | 0.39 |

### Per-Class Metrics (Fine-Tuned Model)
| Label | Precision | Recall | F1-Score | Support |
| :--- | :--- | :--- | :--- | :--- |
| **DD_RESEARCH** | 0.59 | 0.91 | 0.71 | 22 |
| **VIBE_HYPOTHESIS** | 0.00 | 0.00 | 0.00 | 9 |
| **NOISE_MEME** | 0.60 | 0.38 | 0.46 | 8 |

### Confusion Matrix (Test Set)
| True Label \ Predicted Label | DD_RESEARCH | VIBE_HYPOTHESIS | NOISE_MEME |
| :--- | :--- | :--- | :--- |
| **DD_RESEARCH** | 20 | 0 | 2 |
| **VIBE_HYPOTHESIS** | 9 | 0 | 0 |
| **NOISE_MEME** | 5 | 0 | 3 |

## Failure Analysis

The fine-tuned model suffered a 2.6% performance regression compared to the 70-billion parameter baseline. Analyzing the wrong predictions reveals a clear, diagnosable failure mode caused by data distribution constraints.

1.  **Failure 1 (True: VIBE_HYPOTHESIS, Pred: DD_RESEARCH)**
    * *Text:* "[TITLE] Am I supposed to just keep holding forever? When do people actually sell? [BODY] I’m in my early 30s and honestly feel like I’ve gotten more lucky than skilled..."
    * *Analysis:* The model completely failed to predict `VIBE_HYPOTHESIS` across the entire test set. Because the training data was heavily imbalanced toward `DD_RESEARCH`, the linear classification head collapsed. It learned to use `DD_RESEARCH` as a safe default guess whenever it encountered longer paragraphs about personal finance, entirely missing the semantic boundary of a "vibe" post.
2.  **Failure 2 (True: VIBE_HYPOTHESIS, Pred: DD_RESEARCH)**
    * *Text:* "[TITLE] i pulled all my money out of the stock market in 2025 and now with historic highs i can't get back in..."
    * *Analysis:* The model detects terms like "stock market", "money", and "historic highs". Lacking the parameter scale to understand the context of personal regret versus market analysis, the attention mechanism latches onto the financial vocabulary and defaults to the majority class.
3.  **Failure 3 (True: NOISE_MEME, Pred: DD_RESEARCH)**
    * *Text:* "[TITLE] Should I go in and ask for a refund? I wasn’t informed stocks could go down"
    * *Analysis:* This is blatant sarcasm/internet noise. However, because it contains the word "stocks", the DistilBERT model misclassifies it. This highlights a labeling/data problem: the model requires significantly more diverse examples of financial slang and sarcasm to establish a separating hyperplane for `NOISE_MEME`.

## Reflection: Intended vs. Learned Behavior
The intention was to build a model that evaluated the *structural logic* of a post (e.g., does this post contain mathematical reasoning?). Instead, the model learned a shallow heuristic: *if the post mentions money, stocks, or is longer than one sentence, classify it as DD_RESEARCH*. The model severely overfit to the vocabulary of the majority class, completely sacrificing the minority class to minimize its overall Cross-Entropy loss.

## Sample Classifications

| Text Fragment | True Label | Predicted Label | Confidence |
| :--- | :--- | :--- | :--- |
| *"Lululemon downgrades annual outlook. $LULU down 10% post Q1 earnings"* | DD_RESEARCH | NOISE_MEME | 0.40 |
| *"Me coming downstairs after turning $10k Into $800"* | DD_RESEARCH | NOISE_MEME | 0.46 |
| *"Dow tumbles 500 points as possible rate hike under new Fed chief rattles Wall Street"* | NOISE_MEME | DD_RESEARCH | 0.42 |
| *"Apple to raise prices as AI boom pushes up chip costs"* | VIBE_HYPOTHESIS | DD_RESEARCH | 0.60 |

**Correct Prediction Highlight:**
*Text: "Are you guys ready for the absolute bloodbath tomorrow? To the moon lol!"*
* **Predicted:** NOISE_MEME (Confidence: 0.88)
* *Why it is reasonable:* The model successfully identified classic internet slang ("bloodbath", "to the moon", "lol") and correctly grouped it as non-actionable noise, proving it can map specific linguistic markers when they are distinct enough.

## Spec Reflection
The project specification provided a highly structured roadmap for tokenization, preventing out-of-memory errors by establishing dynamic padding and sequence truncation rules. However, I had to diverge from the spec during data ingestion. The raw CSV export contained leading spaces in the headers (e.g., `" label"`). Rather than manually editing the files, I implemented dynamic `.str.strip()` data-cleaning routines in the pandas pipeline to guarantee the code could securely handle raw exports.

## AI Usage
* **Automated Labeling Script:** I directed an LLM to generate a Python script using keyword heuristics to rapidly label 258 scraped Reddit posts. The AI produced a functional loop, but I had to override and manually refine the specific financial keyword triggers (e.g., adding "P/E", "margins") to ensure it mapped accurately to my defined taxonomies.
* **API Output Standardization:** When running the Groq zero-shot baseline, the Llama-3.3 model aggressively injected markdown (asterisks, quotes) into its responses, breaking the strict string-matching parser. I utilized an LLM to debug the classification loop, generating a rapid text-cleaning chain (`.replace("*", "").upper()`) that successfully forced the API outputs into a machine-readable format.