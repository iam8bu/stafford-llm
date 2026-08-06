---
library_name: transformers
license: gpl-3.0
language:
- en
base_model:
- google/gemma-3-4b-it
---

# Model Card for Model ID

🤗 **[View and download STAFFORD on Hugging Face](https://huggingface.co/iam8bu/stafford-llm)**

<!-- Provide a quick summary of what the model is/does. -->

Welcome to STAFFORD (Start/Trade/Add Fantasy Football Optimized Reasoning Decision-maker), a fantasy football LLM trained to answer any start/sit dillema, trade offer, or waiver decision you can come up with. 

## Model Details

### 1. Introduction

I built STAFFORD because I spend way too much time on fantasy football research with mixed results. I am the hardest working manager in my league but have struggled to translate that into wins. I have been experimenting with modeling player performance in recent years in order to improve my outcomes but have found that fantasy football modeling is a uniquely difficult task. The statistics themselves are relatively easy to analyze, but for any fantasy football model worth using, there is a lot of qualitative data that is needed to provide context. For example: a running back may have had good stats last year, but what is missing from the data is that his team drafted a rookie to compete for his job. Or, a quarterback could have had 4 touchdowns last week, but he hurt his hamstring and is expected to be limited going forward. These are scenarios where an LLM could bridge the gap between stats and reality. 

The major commercial models are never specifically trained enough to be truly helpful fantasy football assistants. They get confused about what seasons they are supposed to be looking at and their responses are almost never fully correct. It is too complex and fast-moving of a topic for these generalist models to perform well. I also find that the major LLMs don’t like to commit to an answer. They usually will say something to the effect of “you can’t go wrong with either player.”  I wanted to build a model that is sharp and specially calibrated to provide high-end analysis.

I started off with Gemma 3 4B IT and fine-tuned it using custom NFL training data using LoRA. I benchmarked it using a held-out season of data scored by an AI judge (powered by Gemma 2 2B IT). I found that the fine-tuned model perfomed very well on the benchmark data and vastly outperformed similarly-sized models. Overall, the quality of the model post-fine tuning exceeded my expectations, and I will be using STAFFORD for my fantasy league this season.

### 2. Data

I built my custom training dataset out of real NFL data sourced from NFLverse. I started out by pulling weekly NFL player statistics from the last six seasons (2020-2025) and held out 2025 from my training set for benchmarking. I used Claude Code to help derive some helpful statistics (eg. recent fantasy points/game, boom/bust potential, snap share) and create groups of players that I could later use to simulate realistic prompts for my three categories, start/sit, trade, and waiver wire. 

For start/sit, I generated pairs of similar players at the same position that the model could eventually decide between. For trade, I generated reasonable groups of players that could be traded for one another. For waiver wire, I generated a player to be dropped and three choices to add. For all of these grouped players, I saved a dictionary of helpful stats that could be passed into my prompt generation pipeline. I also simulated a 12 team fantasy football league so that I could pass realistic team-state data to the generation pipeline as well. This enabled me to teach my model risk/reward frameworks on both a weekly and season-long timeline.

I gave this data to Gemma 2 2B and prompted it to generate prompts for each player grouping. I ended up having to do some post processing to insert statistics at the end of each prompt, as Gemma 2 had a habit of dropping them. I then passed these generated prompts to Gemma 3 4B IT using three-shot chain-of-thought prompting to create example responses for each prompt. I did some more post-processing here and stripped out any rambling/hallucinated text to ensure that my training data was clean and ready for fine-tuning. I ended up with 527 examples, split roughly evenly between my three tasks. For train/test split, I used random seed 42 and split the dataset into 423 train, 104 test (~80/20 split). For benchmarking, I used the same generation process on my held-out 2025 data. This dataset contained 100 examples, again split roughly evenly between tasks.

### 3. Methodology

After the training data was ready, I used LoRA and SFT Trainer to train my fantasy football LLM. I used these methods because they performed well in similar tasks I had worked on, and their ability to perform powerful fine tuning efficiently was invaluable given my time and memory constraints. I also knew that LoRA should be able to preserve existing knowledge at least as well as any other method by freezing the pre-trained weights. I used a learning rate of 0.00001 with 3 epochs and 10 eval steps due to my small training dataset. For hyperparameters, I found that an r of 16 and an alpha of 32 worked best after setting them too low initially. Finally, I targeted the full attention layer, which led to large improvements over other configurations I tried. 


### 4. Evaluation

| Model                            | GSM8k-CoT (Flexible) | ARC Challenge (Norm) | Avg Judge Score on 2025 Benchmark Prompts |
|----------------------------------|----------------------|----------------------|-------------------------------------------|
| STAFFORD                         | 0.77                 | 0.57                 | 7.2                                       |
| google/gemma-3-4-it              | 0.77                 | 0.57                 | 8.1                                     |
| microsoft/Phi-4-mini-instruct    | 0.85                 | 0.59                 | 5.0                                       |
| meta-llama/Llama-3.2-3B-Instruct | 0.66                 | 0.46                 | 5.4                                       |

For benchmarking, I used GSM8k-CoT, ARC Challenge, and my own custom fantasy football benchmark mentioned above. I used LM Eval for the first two to assess catastrophic forgetting, because they test math and reasoning that is needed for my fantasy football tasks. For my custom benchmark, I used an AI judge powered by Gemma 2 2B to assess performance on out-of-sample fantasy football prompts. I used the judge to grade the initial responses in my benchmark dataset by giving it a robust prompt with performance categories to think through and scoring buckets to conform to. I had it generate a score out of 10, with 10 being a perfect score. I then stripped that score out of the review the AI judge generated and added it as a new column to my benchmark. I then ran the benchmark prompts through my fine-tuned model and scored the results with the AI judge. I took the average of the AI judge scores for each model to compare response quality.

My AI judge may have graded the trained responses slightly harsher than the initial responses, but I see a more nuanced picture when looking at the actual results. First of all, when I generated these responses with the untrained Gemma 3, I had to trim hallucinatory text from the end of almost every single response. I had to do this in order to generate a good set of data, but the AI Judge scores would be much worse if I hadn't trimmed them. The fact that my zero-shot, fine-tuned model is close to the edited, 3-shot training responses is very positive. Further, you would expect a model to perform slightly worse on out-of-sample data, so I still read this as encouraging. 

Second, I actually like a lot of the responses I looked at better from the trained model than the untrained. The trained model looks a lot more like a commercial LLM's response. They are bulleted, flow nicely, and extrapolate more on the statistics given. There may be some quirks with my AI judge that don't quite capture what I was looking for. Additionally, the fine tuning led to very little catastrophic forgetting, as it performed essentially the same on the two lm eval benchmarks.

I also ran Microsoft's Phi 4 and Meta's Llama 3.2 through the same benchmarks to compare to STAFFORD. I chose these because they are of similar size, were created for similar purposes as Gemma 3, and are from two other large companies. As you can see from the table above, those models scored much worse on my 2025 NFL benchmark than STAFFORD did. 

### 5. Usage and Intended Uses

```{python}
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch
from peft import PeftModel

tokenizer = AutoTokenizer.from_pretrained("google/gemma-3-4b-it")
base_model = AutoModelForCausalLM.from_pretrained("google/gemma-3-4b-it", device_map="auto", dtype=torch.bfloat16)
model = PeftModel.from_pretrained(base_model, "iam8bu/stafford-llm")
```

STAFFORD was built to make decisions on three categories: start/sit decisions, trade offers, and waiver add/drops. It was trained on PPR data and simulated a 12 team redraft league for risk/reward training. Treat these as default values, any variations in your league should be included in your prompt so that STAFFORD will take them into account.

Start/Sit: The training data included decisions for every position combination, including flex spots. It did not take into account superflex leagues, so QB vs other positions may not function as well. Defenses and kickers were not part of the training data.

Trade Offers: The training data included 1 for 1, 2 for 1, and 2 for 2 trades. Larger trades or trades in dynasty leagues may not function as well.

Waiver: The model was trained on scenarios where one player would be dropped and three options for replacing that player would be considered. Waiver budget was not part of the training data and the model may not be able to answer related questions as well. 

STAFFORD was not trained on other types of questions such as draft planning, player projections, or other fantasy topics, it may be able to generalize the data that it has learned to those types of questions, but take its opinions with a grain of salt.

### 6. Prompt Format

For all prompts, include as much data about the players considered, your team's performance, and your league status as possible. Each piece of data will improve STAFFORD's ability to tailor a response to your unique situation. For example:

```
"I am deciding between starting Travis Kelce and Harold Fannin Jr. in my TE spot this week. Kelce has been consitently scoring about 10 points per game the last four weeks but has not scored more than 15 all season. He is facing the Broncos this week which is a tough matchup. Fannin is also averaging 10 points but is more up and down, some weeks he will score only 5 points and others he will score 20. He has an easier matchup but the Browns are big favorites so I am concerned they will pack it in early. I am 4-4 and projected to lose this week. I could really use a win to get me closer to the playoffs."
```

### 7. Expected Response

The model should take all of your datapoints into consideration, give a 2-3 sentence, bulleted breakdown of each player, and then give a clear, 2-3 sentence recommendation that takes into account player info and your situation. I would recommend setting the max tokens in your pipe to 300-400 to avoid cutting off STAFFORD's response.

```
This is a tough call, and you're right to be considering the risk versus reward. Here's a breakdown to help you decide:

**Arguments for Kelce:**

*   **Consistency:** You've seen him consistently score around 10 points, which is valuable in a week where you need a boost.
*   **Higher Floor:** Even with a tough matchup, he's more likely to provide a solid, predictable 10 points.

**Arguments for Fannin:**

*   **Higher Ceiling:** He has the potential for a huge game (20 points), which is exactly what you need to swing a close game.
*   **Easier Matchup:** While the Browns are big favorites, a blowout win could lead to him getting more opportunities.

**Recommendation:**

Given your 4-4 record and need for a win, you need to take a calculated risk. **I'm leaning towards Fannin Jr.** The potential for a massive game outweighs the risk of a low-scoring performance. A 20-point game could be the difference between a win and a loss, and you're already projected to lose. 

**However,** this is a gamble. If Fannin scores only 5 points, you'll be frustrated. But if he explodes, you'll be ecstatic.

**Final Decision:** Start Harold Fannin Jr.
```

### 8. Limitations

My model has several limitations:

1. The training dataset was small and limited to only three scenarios. In the interest of time and memory constraints, I trained my model with a relatively small dataset. In a perfect world, I could have added more scenarios, such as draft info, broader fantasy strategy information, and player projections. I could also have used a broader range of data to gather more datapoints for the scenarios I did have.
2. Using untrained models to generate training prompts and responses is not ideal. These models were not fine-tuned for fantasy football analysis, and the training data definitely had some hiccups as a result. Using STAFFORD to train my next model may yield better results.
3. The prompts are only a rough approximation of what an actual fantasy football manager would ask. Most fantasy managers would default to a simple, one-line prompt such as "should I start player A or Player B this week." STAFFORD does not have up-to-date NFL statistics, so a user must insert all statistics they wish the model to consider into their prompt. A RAG pipeline or internet lookup capability would be worthwile.
4. The model will not be able to update NFL knowledge without further fine tuning. As a consequence, it will not be able to adjust to trends that emerge in the next few years, and its backround knowledge may go out of date.

### References
Aggie Sports Analytics. (2024). FantasyChatbot [Computer software]. GitHub. https://github.com/AggieSportsAnalytics/FantasyChatbot

Anthropic. (2026). Claude [Large language model]. https://www.anthropic.com

Carl, S., & Baldwin, B. (2023). nflfastR: Functions to efficiently access NFL play by play data (R package version 4.6.0). https://www.nflfastr.com/

Google DeepMind. (n.d.-a). google/gemma-2-2b-it. Hugging Face. Retrieved August 5, 2026, from https://huggingface.co/google/gemma-2-2b-it

Google DeepMind. (n.d.-b). google/gemma-3-4b-it. Hugging Face. Retrieved August 5, 2026, from https://huggingface.co/google/gemma-3-4b-it

Hu, E. J., Shen, Y., Wallis, P., Allen-Zhu, Z., Li, Y., Wang, S., Wang, L., & Chen, W. (2021). LoRA: Low-rank adaptation of large language models. arXiv. https://arxiv.org/abs/2106.09685

Hugging Face. (n.d.). LoRA. PEFT Documentation. Retrieved August 5, 2026, from https://huggingface.co/docs/peft/main/en/conceptual_guides/lora

LLM Stats. (n.d.). GSM8K leaderboard. Retrieved August 5, 2026, from https://llm-stats.com/benchmarks/gsm8k

nflverse. (n.d.). nflverse-data [Data set]. GitHub. Retrieved August 5, 2026, from https://github.com/nflverse/nflverse-data/releases

Tschand, A. (n.d.). FantasyFootballBench: Evaluating LLM negotiation and long-horizon decision making with fantasy football. https://www.aryatschand.com/blog/blog-fantasyfootballbench.html

Unsloth AI. (n.d.). LoRA hyperparameters guide. Unsloth Documentation. Retrieved August 5, 2026, from https://docs.unsloth.ai/get-started/fine-tuning-guide/lora-hyperparameters-guide

Wei, J., Wang, X., Schuurmans, D., Bosma, M., Ichter, B., Xia, F., Chi, E., Le, Q., & Zhou, D. (2022). Chain-of-thought prompting elicits reasoning in large language models. arXiv. https://arxiv.org/abs/2201.11903

Note - I used Claude for brainstorming, to help get my raw data into a usable format, to help me implement an AI judge in Python, to help fit the code from class to my unique dataset, and for some trouble-shooting and error handling when internet sources were not helpful.
