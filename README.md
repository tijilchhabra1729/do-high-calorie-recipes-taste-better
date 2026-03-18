---
layout: default
title: Do Higher-Calorie Recipes Taste Better?
---

# Do Higher-Calorie Recipes Taste Better? 🍽️
## Investigating the Relationship Between Calories and Ratings on Food.com

**By: Tijil Chhabra**

*A DSC 80 Final Project at UC San Diego*

---

## Introduction

This project investigates the **Recipes and Ratings** dataset from [food.com](https://www.food.com/), originally scraped by Bodhisattwa Prasad Majumder et al. for their recommender systems research. The dataset contains recipes and user reviews posted since 2008.

**Central Question:** *Do recipes with higher calorie counts tend to receive different average ratings than recipes with lower calorie counts?*

This question is relevant because it explores whether indulgent, calorie-dense recipes genuinely earn better reviews from users or if lighter, healthier recipes are just as well-received. Understanding this relationship can help recipe developers, food bloggers, and health-conscious cooks make informed decisions.

The raw data comes in two CSV files:
- `RAW_recipes.csv`: **83,782 unique recipes** with 12 columns.
- `RAW_interactions.csv`: **731,927 reviews** with 5 columns.

Key columns relevant to our analysis:

| Column | Description |
|--------|-------------|
| `name` | Recipe name |
| `id` | Recipe ID |
| `minutes` | Minutes to prepare the recipe |
| `n_steps` | Number of steps in the recipe |
| `n_ingredients` | Number of ingredients in the recipe |
| `nutrition` | Nutrition info as a string: `[calories (#), total fat (PDV), sugar (PDV), sodium (PDV), protein (PDV), saturated fat (PDV), carbohydrates (PDV)]` |
| `rating` | Rating given by user (1–5), from the interactions dataset |
| `description` | User-provided description of the recipe |

---

## Data Cleaning and Exploratory Data Analysis

### Cleaning Steps

1. **Left-merged** recipes with interactions on `id`/`recipe_id` so every recipe retains its row even if it has zero reviews. The merged dataset has **234,429 rows**.
2. **Replaced 0 ratings with `NaN`** — on food.com, the lowest possible rating is 1 star. A rating of 0 means the user wrote a review but chose not to leave a star rating, so it represents *missing data*, not an actual score.
3. **Computed average rating per recipe** by grouping by recipe `id` and averaging non-null ratings, giving one aggregate score per recipe.
4. **Parsed the `nutrition` column** from a string (e.g. `'[138.4, 10.0, 50.0, ...]'`) into seven individual numeric columns: `calories`, `total_fat_pdv`, `sugar_pdv`, `sodium_pdv`, `protein_pdv`, `saturated_fat_pdv`, `carbohydrates_pdv`.
5. **Created `calorie_category`** — High Calorie (above median of **305.4 calories**) or Low Calorie — for use in hypothesis testing.
6. **Created `rating_category`** — High (avg_rating > 4) or Low (avg_rating ≤ 4) — as the response variable for the classification task. **2,609 recipes** have missing `avg_rating` (no reviews at all).

Here are the first few rows of the cleaned dataset:

| name                                     | id     | minutes | n_steps | n_ingredients | calories | avg_rating | calorie_category | rating_category |
|------------------------------------------|--------|---------|---------|---------------|----------|------------|------------------|-----------------|
| 1 brownies in the world best ever        | 333281 | 40      | 10      | 9             | 138.4    | 4.0        | Low Calorie      | Low             |
| 1 in canada chocolate chip cookies       | 453467 | 45      | 12      | 11            | 595.1    | 5.0        | High Calorie     | High            |
| 412 broccoli casserole                   | 306168 | 40      | 6       | 9             | 194.8    | 5.0        | Low Calorie      | High            |
| millionaire pound cake                   | 286009 | 120     | 7       | 7             | 878.3    | 5.0        | High Calorie     | High            |
| 2000 meatloaf                            | 475785 | 90      | 17      | 13            | 267.0    | 5.0        | Low Calorie      | High            |

### Univariate Analysis

<iframe
  src="assets/ratings_distribution.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

The distribution of average ratings is **heavily left-skewed**, with the vast majority of recipes rated near 5.0. This is typical of voluntary review systems — users who are satisfied are more likely to leave a rating, while those who are indifferent or mildly dissatisfied may not rate at all.

<iframe
  src="assets/calories_distribution.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

The calorie distribution is **right-skewed**: most recipes are below 500 calories, with a long tail of high-calorie outliers. This is why we used a median-based split for grouping recipes into High and Low calorie categories.

### Bivariate Analysis

<iframe
  src="assets/rating_by_calorie_box.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Both calorie groups have similar median ratings (close to 5.0), but the High Calorie group shows slightly more spread in the lower quartiles — a subtle difference worth testing formally.

<iframe
  src="assets/cal_vs_rating_scatter.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

The scatter plot shows **no strong linear trend** between calories and rating. Recipes across the entire calorie spectrum cluster near 4–5 stars, consistent with the left-skewed rating distribution.

### Interesting Aggregates

| calorie_bin | 1–5 steps | 6–10 steps | 11–20 steps | 20+ steps |
|-------------|-----------|------------|-------------|-----------|
| 0–100       | 4.6697    | 4.6278     | 4.6316      | 4.6140    |
| 101–250     | 4.6461    | 4.6189     | 4.6191      | 4.6663    |
| 251–500     | 4.6256    | 4.6124     | 4.6187      | 4.6293    |
| 501–1000    | 4.5920    | 4.6128     | 4.6366      | 4.6641    |
| 1000+       | 4.6203    | 4.6143     | 4.6123      | 4.6382    |

Average ratings are remarkably stable (~4.6–4.7) across all calorie ranges and step counts. Recipes with 501–1000 calories and 1–5 steps show a slight dip to 4.59, but overall, neither calorie count nor complexity has a dramatic impact on ratings.

---

## Assessment of Missingness

### NMAR Analysis

The `rating` column in the interactions dataset is likely **NMAR** (Not Missing At Random). On food.com, users can submit text reviews without leaving a star rating — in the raw data these appear as 0, which we replaced with `NaN`. The decision to skip the star rating likely depends on the rating value the user *would have* given: users with neutral or lukewarm opinions may not feel strongly enough to assign a number, while users with very strong positive or negative feelings are more motivated to rate. Since the probability of missingness depends on the unobserved value itself, this is **NMAR**.

To potentially make this MAR, we could collect additional data such as whether the user actually cooked the recipe, time spent on the recipe page, or whether the user saved/bookmarked the recipe. If the missingness could be fully explained by such observable variables, it would become MAR.

### Missingness Dependency

**Test 1: Does missingness of `avg_rating` depend on `n_steps`?**

We performed a permutation test with 1,000 iterations, using the absolute difference in mean `n_steps` as the test statistic and a significance level of α = 0.05. The observed difference was **1.4930**.

<iframe
  src="assets/missingness_nsteps.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

**Result:** The p-value is **0.0**, so we **reject the null hypothesis**. The missingness of `avg_rating` **depends on** `n_steps`. Recipes with more steps tend to have different rates of missing ratings — likely because complex recipes attract fewer total reviews.

**Test 2: Does missingness of `avg_rating` depend on `sodium_pdv`?**

We performed a permutation test with 1,000 iterations using the Kolmogorov-Smirnov (K-S) statistic — the maximum absolute difference between the two groups' CDFs. We chose the K-S statistic over difference in means because with 83,000+ rows, even trivially small differences in means can appear statistically significant; the K-S statistic better measures whether the two distributions actually differ in shape. The observed K-S statistic was **0.0237**.

<iframe
  src="assets/missingness_sodium_perm.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

**Result:** The p-value is **0.087**, which is greater than α = 0.05, so we **fail to reject the null hypothesis**. The missingness of `avg_rating` **does not appear to depend on** `sodium_pdv`. A recipe's sodium content does not significantly predict whether it will have missing ratings.

---

## Hypothesis Testing

**Question:** Do recipes with higher calorie counts have different average ratings than lower-calorie recipes?

We split recipes into two groups at the median calorie count (305.4 calories):
- **High Calorie:** calories > 305.4
- **Low Calorie:** calories ≤ 305.4

**Null Hypothesis:** Recipes with higher calorie counts and recipes with lower calorie counts have the same average rating. Any observed difference is due to random chance.

**Alternative Hypothesis:** Recipes with higher calorie counts have a *different* average rating than recipes with lower calorie counts.

**Test Statistic:** Difference in group means (High Calorie − Low Calorie).

**Significance Level:** α = 0.05

We chose a permutation test because we are comparing two observed groups from the same dataset. The difference in means is natural for comparing a quantitative variable across two groups, and a two-sided test is appropriate since we have no prior directional expectation.

**Observed results:**
- Mean rating (High Calorie): **4.6213**
- Mean rating (Low Calorie): **4.6294**
- Observed difference: **−0.0082**

<iframe
  src="assets/hypothesis_test.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

**Result:** The p-value is **0.0721**, which is greater than α = 0.05, so we **fail to reject the null hypothesis**. We do not have sufficient evidence to conclude that calorie content is associated with a difference in average ratings. The tiny observed difference of −0.008 rating points is consistent with random variation. This is an observational study, so even if we had rejected the null, we could not have concluded causation.

---

## Framing a Prediction Problem

**Problem:** Predict whether a recipe will receive a **High** (avg_rating > 4) or **Low** (avg_rating ≤ 4) average rating.

**Type:** Binary classification.

**Response Variable:** `rating_category` — chosen because the raw `avg_rating` distribution is extremely left-skewed (most recipes are ~5 stars), making exact numeric prediction very difficult. Binarizing at 4 creates a more balanced and interpretable task.

**Metric:** F1-score (weighted), chosen over accuracy because the classes are imbalanced — **63,606 High** vs. **17,567 Low** (78% vs. 22%). A model that always predicts "High" would achieve 78% accuracy but be useless. F1-score penalizes models that ignore the minority class.

**Features at time of prediction:** Only recipe attributes known before any ratings exist: `calories`, `n_steps`, `n_ingredients`, `minutes`, `sugar_pdv`, `protein_pdv`, `sodium_pdv`, `total_fat_pdv`, `saturated_fat_pdv`, `carbohydrates_pdv`. We do NOT use `rating`, `avg_rating`, `review`, or `date`.

---

## Baseline Model

**Model:** Decision Tree Classifier (default hyperparameters).

**Features (2):**
1. `n_ingredients` — **quantitative**. Reflects recipe complexity. Left as-is (passthrough) since decision trees handle numeric features natively.
2. `calorie_category` — **nominal** (High Calorie / Low Calorie). Encoded with `OneHotEncoder(drop='if_binary')`, creating a single binary column.

**Performance:**
- Training F1 (weighted): **0.6885**
- Test F1 (weighted): **0.6885**
- Test Accuracy: **0.7836**

This baseline model is **not good**. While it achieves 78% accuracy, this is misleading — the model predicts "High" for nearly every recipe and completely ignores the "Low" class (0.00 recall, 0.00 F1 for "Low"). This is a consequence of class imbalance: about 78% of recipes are rated above 4, so a naive model always predicting "High" achieves similar accuracy. The identical training and test scores suggest the model is underfitting — with only two features, it cannot learn meaningful patterns.

---

## Final Model

**Algorithm:** Random Forest Classifier with `class_weight='balanced'`.

**New Features Added (beyond baseline):**
- `calories` (StandardScaler) — the continuous calorie count provides much finer-grained information than the binary High/Low split. A 200-calorie recipe and a 400-calorie recipe were both "Low Calorie" in the baseline but may have very different rating patterns.
- `sugar_pdv` (QuantileTransformer) — sugar content is heavily right-skewed with extreme outliers. The quantile transformation maps it to a uniform distribution, preventing a few extreme-sugar recipes from dominating tree splits. Sugar also captures recipe type — desserts and baked goods have different rating distributions than savory dishes.
- `protein_pdv` (StandardScaler) — protein signals whether a recipe is a hearty main dish (high protein) or a snack/dessert (low protein), adding a nutritional dimension the baseline lacked.
- `n_steps` (passthrough) — reflects recipe complexity, which correlates with cook experience and rating behavior.
- `minutes` (StandardScaler) — preparation time captures a dimension of difficulty not reflected by step count alone.

**Hyperparameter Tuning:** `GridSearchCV` with 5-fold cross-validation over `max_depth` ∈ {5, 10, 20}, `min_samples_split` ∈ {10, 50, 100}, `n_estimators` ∈ {100, 200}.

**Best Hyperparameters:** `max_depth=20`, `min_samples_split=10`, `n_estimators=200`.

**Performance:**
- Training F1 (weighted): **0.9576**
- Test F1 (weighted): **0.6931** (improvement of **+0.0046** over baseline)
- Test Accuracy: **0.7307**

The key improvement was adding `class_weight='balanced'`, which forces the Random Forest to penalize misclassification of "Low" recipes more heavily. This dramatically improved Low recall from 0% to 12% and Low F1 from 0.00 to 0.16. While the overall improvement in weighted F1 is modest (+0.0046), the model now meaningfully identifies some low-rated recipes instead of blindly predicting "High" for everything. This is a fundamentally difficult prediction task — the vast majority of recipes on food.com are rated above 4 stars, leaving limited signal to distinguish the minority class from recipe attributes alone.

<iframe
  src="assets/confusion_matrix.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

---

## Fairness Analysis

**Question:** Does our model perform equally well for high-calorie vs. low-calorie recipes?

- **Group X:** High-Calorie recipes (calories > 305.4)
- **Group Y:** Low-Calorie recipes (calories ≤ 305.4)
- **Evaluation Metric:** F1-score (weighted)
- **Null Hypothesis:** Our model is fair. Its F1-score for High-Calorie and Low-Calorie recipes are roughly the same, and any differences are due to random chance.
- **Alternative Hypothesis:** Our model is unfair. Its F1-score differs between the two calorie groups.
- **Test Statistic:** Absolute difference in F1-scores.
- **Significance Level:** α = 0.05

**Observed Results:**
- F1 (High Calorie): **0.6792**
- F1 (Low Calorie): **0.7060**
- Observed |F1 difference|: **0.0269**

<iframe
  src="assets/fairness_test.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

**Result:** The p-value is **0.0**, which is less than α = 0.05, so we **reject the null hypothesis**. Our model appears to be unfair — it performs significantly differently for high-calorie recipes (F1 = 0.679) compared to low-calorie recipes (F1 = 0.706). This is not entirely surprising since calorie content is one of the model's features, so the model may have learned different decision boundaries for each group. Additionally, the class balance (proportion of High vs. Low rated recipes) may differ between the two calorie groups, contributing to unequal performance.
