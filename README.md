# Does First Pick Win? Analyzing Blue Side Advantage in Professional League of Legends

**By: Aaditya Aswadhati**

## Introduction

This project analyzes professional League of Legends esports match data from 2025 to investigate whether **draft order provides a competitive advantage**. Specifically, we ask: **Does Blue side (which picks first in the draft) win more often than Red side?**

In League of Legends, the draft phase determines which champions each team will play. Blue side always gets the first pick, while Red side gets the last pick and counter-pick opportunities. This tradeoff raises the question: does picking first actually translate to winning more games?

This question matters because if a significant Blue side advantage exists, it could indicate an imbalance in the game's competitive format that Riot Games (the developer) may need to address.

**Dataset Overview:**
- **Rows:** 119,556 (12 rows per game: 5 players + 1 team summary per side)
- **Relevant Columns:**
  - `gameid`: Unique identifier for each match
  - `side`: Which side the team played on ("Blue" or "Red") - Blue side picks first
  - `result`: Whether the team won (1) or lost (0)
  - `pick1`-`pick5`: The champions picked by each team (in draft order)
  - `ban1`-`ban5`: The champions banned by each team

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

The raw dataset contains 12 rows per game: 10 player rows (5 per team) and 2 team summary rows. For our analysis of Blue side vs Red side win rates, we separated the data into player-level and team-level DataFrames. We primarily use the team-level data since each game has exactly one Blue team row and one Red team row.

We also converted boolean columns (`result`, `playoffs`, `firstblood`, etc.) from integers/floats to proper boolean types for cleaner analysis.

| gameid | side | result | teamname | pick1 | pick2 | pick3 | pick4 | pick5 |
|--------|------|--------|----------|-------|-------|-------|-------|-------|
| LOLTMNT03_179647 | Blue | 0 | IziDream | Rumble | Viego | Ahri | Jinx | Leona |
| LOLTMNT03_179647 | Red | 1 | Team Valiant | Ambessa | Xin Zhao | Syndra | Ashe | Nautilus |

### Univariate Analysis

We examined the distribution of wins by side across all professional games in 2025.

<iframe src="assets/win_rate_by_side.html" width="800" height="600" frameborder="0"></iframe>

The bar chart shows that Blue side has a slightly higher win rate than Red side (approximately 51.5% vs 48.5%), suggesting a potential first-pick advantage.

### Bivariate Analysis

We examined whether the Blue side advantage varies by game duration.

<iframe src="assets/win_rate_by_side_duration.html" width="800" height="600" frameborder="0"></iframe>

The grouped bar chart reveals that Blue side's advantage is relatively consistent across different game lengths, though it may be slightly more pronounced in shorter games.

### Interesting Aggregates

We analyzed win rates by side across different professional leagues to see if the Blue side advantage is consistent globally.

| League | Blue Win Rate | Red Win Rate | Blue Advantage |
|--------|---------------|--------------|----------------|
| LCK | 0.532 | 0.468 | 0.064 |
| LEC | 0.521 | 0.479 | 0.042 |
| LCS | 0.518 | 0.482 | 0.036 |
| LPL | 0.515 | 0.485 | 0.030 |

The Blue side advantage exists across all major leagues, though the magnitude varies.

---

## Assessment of Missingness

### NMAR Analysis

I believe the missingness in **pick columns** could be **NMAR** (Not Missing At Random). If a game had technical issues or was remade, the draft data might not be recorded, and the reason for the missing data (technical problems) is related to the game itself not being completed properly - information not captured in other columns.

To make this MAR, we would need additional data such as whether games were remade or had technical pauses during the draft phase.

### Missingness Dependency

We analyzed the missingness of `pick1` (first champion pick) which has 38 missing values (0.19%).

**Test 1: Does pick1 missingness depend on league?**

We performed a permutation test using Total Variation Distance (TVD) as our test statistic.

<iframe src="assets/missingness_league_test.html" width="800" height="600" frameborder="0"></iframe>

- Observed TVD: 0.7038
- P-value: 0.0

**Conclusion:** The missingness of pick1 **depends on** league (MAR). Different leagues have different data collection practices.

**Test 2: Does pick1 missingness depend on side?**

- Blue side missing rate: 0.19%
- Red side missing rate: 0.19%
- Observed difference: 0.0000
- P-value: 1.0

**Conclusion:** The missingness of pick1 **does not depend on** side. Whether a team is Blue or Red has no effect on data collection.

---

## Hypothesis Testing

We formally tested whether Blue side has a statistically significant advantage over Red side.

**Null Hypothesis (H₀):** Blue side and Red side have the same win rate. Any observed difference is due to random chance.

**Alternative Hypothesis (H₁):** Blue side has a higher win rate than Red side.

**Test Statistic:** Difference in win rates (Blue win rate - Red win rate)

**Why this test statistic?** The difference in win rates directly measures the quantity we care about - whether Blue side wins more often. It's intuitive, interpretable, and captures the essence of our research question.

**Significance Level:** α = 0.05

**Why α = 0.05?** This is the conventional threshold in statistical analysis, balancing the risk of false positives (claiming an advantage exists when it doesn't) against false negatives (missing a real advantage). For a competitive gaming context, this threshold is appropriate since the consequences of either error are not severe.

**Why a permutation test?** We chose a permutation test over a parametric test (like a z-test for proportions) because:
1. It makes no assumptions about the underlying distribution of win rates
2. It directly simulates the null hypothesis by randomly shuffling side labels
3. It's robust and intuitive - if side truly doesn't matter, shuffling labels shouldn't change the observed difference

**Results:**
- Blue side win rate: 51.5%
- Red side win rate: 48.5%
- Observed difference: 3.0 percentage points
- P-value: 0.0

<iframe src="assets/hypothesis_test.html" width="800" height="600" frameborder="0"></iframe>

**Conclusion:** Since the p-value (≈0) is less than our significance level (0.05), we **reject the null hypothesis**. There is statistically significant evidence that Blue side has a higher win rate than Red side in professional League of Legends. This suggests that the first-pick advantage in the draft phase translates to a measurable competitive edge.

---

## Framing a Prediction Problem

**Prediction Problem:** Predict whether a team will win or lose a League of Legends match based solely on draft information.

**Type:** Binary Classification (Win = 1, Loss = 0)

**Response Variable:** `result` - Whether the team won the game. We chose this because it directly measures competitive success and ties into our earlier finding that Blue side has an advantage.

**Features (known at time of prediction):** All draft information is known before the game starts:
- `side` - Blue or Red (which determines pick order)
- `pick1` - `pick5` - The 5 champions selected by the team
- `ban1` - `ban5` - The 5 champions banned by the team

**Evaluation Metric:** Accuracy. We chose accuracy because:
1. Our classes are balanced (every game has exactly one winner and one loser, so 50% baseline)
2. There's no asymmetric cost to false positives vs false negatives
3. It's interpretable - "what percentage of games do we correctly predict?"

---

## Baseline Model

Our baseline model is a **Logistic Regression classifier** that predicts game outcome using two features:

**Features:**
- `side` (nominal) - One-hot encoded to 1 binary column
- `pick1` (nominal) - One-hot encoded to ~150 columns (one per unique champion)

**Performance:**
- Training Accuracy: 54.32%
- Test Accuracy: 51.86%

**Assessment:** Since wins and losses are perfectly balanced (50%), an accuracy of 50% represents random guessing. Our baseline model achieves 51.86% on unseen test data - only marginally better than random chance. This is expected because we're only using two features: side (which provides a small advantage) and a single champion pick (which doesn't capture full team composition). The model has significant room for improvement.

---

## Final Model

We improved upon our baseline by adding more features, engineering new features based on domain knowledge, and tuning hyperparameters.

### Features Added

**All Picks (`pick1`-`pick5`):** The baseline only used the first pick, but a team's strength comes from their entire composition. Different champions synergize with each other - for example, a team with strong engage champions needs follow-up damage dealers. By including all five picks, the model can learn which champion combinations tend to win more often.

**All Bans (`ban1`-`ban5`):** Bans reveal strategic information about what teams fear playing against. If a team bans five assassins, it suggests their composition may be vulnerable to burst damage. Including ban information allows the model to capture these strategic considerations that exist before the game even starts.

### Engineered Features

**`num_meta_picks` (count of top-20 most picked champions on the team):** 
From the perspective of the data generating process, "meta" champions are heavily picked because professional analysts and coaches have identified them as strong in the current patch. Teams that draft more meta champions are leveraging collective professional knowledge about what's effective. A team with 4-5 meta picks is playing "standard" while a team with 0-1 meta picks is likely running an unconventional strategy that may be riskier.

**`num_meta_bans` (count of top-20 most banned champions in the team's bans):**
Similarly, the most banned champions are those the professional community considers most threatening or overpowered. Teams that ban meta champions are engaging in standard strategic play, while teams that use bans on non-meta champions may be target-banning specific players or hiding their own strategy. This feature captures how conventionally a team is approaching the draft.

### Modeling Algorithm

We tested both **Random Forest** and **Logistic Regression** with the same features:

| Model | Training Accuracy | Test Accuracy |
|-------|------------------|---------------|
| Random Forest | 65.79% | 53.82% |
| Logistic Regression | 59.02% | 54.43% |

We selected **Logistic Regression** as our final model because:
1. It achieved higher test accuracy (54.43% vs 53.82%)
2. It showed less overfitting - the gap between training and test accuracy was smaller (4.6% vs 12%)
3. The relationship between draft features and winning appears to be relatively linear, so a simpler model generalizes better

### Hyperparameter Tuning

We used **GridSearchCV with 5-fold cross-validation** to tune hyperparameters, testing:
- `C` (regularization strength): [0.01, 0.1, 1, 10]
- `penalty`: ['l2']

**Best Hyperparameters:** C=1, penalty='l2'

The regularization strength C=1 provides moderate regularization, preventing the model from overfitting to noise in the high-dimensional one-hot encoded champion features while still allowing it to learn meaningful patterns.

### Performance Improvement

| Metric | Baseline | Final Model | Improvement |
|--------|----------|-------------|-------------|
| Test Accuracy | 51.86% | 54.43% | +2.57 percentage points |
| Training Accuracy | 54.32% | 59.02% | +4.70 percentage points |

The Final Model achieves 54.43% test accuracy compared to the baseline's 51.86%, an improvement of 2.57 percentage points (4.96% relative improvement).

**Why the improvement is modest:** Predicting game outcomes from draft alone is inherently difficult. While draft matters, professional League of Legends outcomes are heavily influenced by factors not captured in draft data: individual player skill, team coordination, in-game decision making, and day-to-day performance variance. The fact that we can predict better than random chance (50%) demonstrates that draft information does contain signal, but the ceiling is likely limited. Our model successfully extracts what predictive value exists in draft data.

---

## Fairness Analysis

We analyzed whether our model performs fairly across different groups.

**Question:** Does our model perform differently for Blue side versus Red side?

**Groups:**
- Group X: Blue side teams
- Group Y: Red side teams

**Evaluation Metric:** Accuracy

**Hypotheses:**
- **H₀:** The model is fair. Accuracy is the same for Blue and Red side, and any differences are due to random chance.
- **H₁:** The model is unfair. Accuracy differs between Blue and Red side.

**Test Statistic:** Absolute difference in accuracy

**Significance Level:** α = 0.05

**Results:**
- Blue side accuracy: 53.44%
- Red side accuracy: 55.44%
- Observed difference: 2.00 percentage points
- P-value: 0.202

<iframe src="assets/fairness_test.html" width="800" height="600" frameborder="0"></iframe>

**Conclusion:** Since the p-value (0.202) is greater than our significance level (0.05), we fail to reject the null hypothesis. There is not enough statistical evidence to conclude that our model performs differently for Blue side versus Red side teams. **The model appears to be fair with respect to side.**