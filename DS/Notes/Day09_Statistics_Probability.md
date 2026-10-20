# Day 9: Statistics & Probability for Data Science Engineers
## Descriptive Metrics, Bayes' Theorem, Distributions, Hypothesis Testing, A/B Testing & Executive Translation

---

# Part 1: Descriptive Statistics & Measures of Dispersion

### 1.1 Central Tendency & Distribution Skewness

| Metric | Mathematical Definition | Resilience to Outliers | Best Used When |
| :--- | :--- | :--- | :--- |
| **Mean ($\mu, \bar{x}$)** | $\frac{1}{N}\sum_{i=1}^N x_i$ | **Zero Resilience** (Pulled heavily by extreme values) | Data is symmetric and normally distributed without extreme outliers. |
| **Median** | Middle value of sorted data ($50^{\text{th}}$ percentile) | **High Resilience** | Data is skewed (e.g., Household income, Customer LTV, Website latency). |
| **Mode** | Most frequently occurring value | **Immune** | Categorical data or multi-modal distributions. |

```
Positive (Right) Skew:       Mode < Median < Mean   (e.g., Salaries, Page response times)
Symmetric (Normal):          Mode = Median = Mean   (e.g., Human height, Measurement errors)
Negative (Left) Skew:        Mean < Median < Mode   (e.g., Age at retirement, Exam scores)
```

---

### 1.2 Measures of Dispersion & Bessel's Correction
- **Population Variance ($\sigma^2$):** $\sigma^2 = \frac{1}{N}\sum_{i=1}^N (x_i - \mu)^2$
- **Sample Variance ($s^2$):** $s^2 = \frac{1}{n-1}\sum_{i=1}^n (x_i - \bar{x})^2$

#### Why divide by $n-1$ (Bessel's Correction)?
Because the sample mean $\bar{x}$ is estimated from the exact same sample data, the deviations $(x_i - \bar{x})$ are systematically slightly smaller than the deviations from the true population mean $\mu$. Dividing by $n$ produces an underestimate (**biased estimator**). Dividing by $n-1$ removes the bias, making $s^2$ an **unbiased estimator** of $\sigma^2$.

- **Standard Deviation ($\sigma, s$):** Square root of variance. Expressed in the **exact same units** as the underlying data.
- **Interquartile Range (IQR):** $\text{IQR} = Q_3 - Q_1$ ($75^{\text{th}}$ percentile minus $25^{\text{th}}$ percentile). Measures spread of the central 50% of data; immune to extreme tails.

---

# Part 2: Probability Foundations & Bayesian Inference

### 2.1 Conditional Probability & Multiplication Rule
The conditional probability of event $A$ occurring given that event $B$ has already occurred is:
$$P(A|B) = \frac{P(A \cap B)}{P(B)} \quad (\text{provided } P(B) > 0)$$

Two events $A$ and $B$ are **statistically independent** if and only if:
$$P(A \cap B) = P(A) \cdot P(B) \iff P(A|B) = P(A)$$

---

### 2.2 The Law of Total Probability & Bayes' Theorem
Given mutually exclusive and exhaustive partitions $B_1, B_2, \dots, B_k$:
$$P(A) = \sum_{i=1}^k P(A|B_i)P(B_i)$$

**Bayes' Theorem** updates prior beliefs in the presence of new evidence:
$$P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B)} = \frac{P(B|A) \cdot P(A)}{\sum_{i} P(B|A_i)P(A_i)}$$

- $P(A)$: **Prior Probability** (Belief before observing evidence $B$).
- $P(B|A)$: **Likelihood** (Probability of observing evidence $B$ if hypothesis $A$ is true).
- $P(B)$: **Marginal Evidence** (Total probability of observing evidence $B$).
- $P(A|B)$: **Posterior Probability** (Updated belief after observing evidence $B$).

---

### 2.3 The Classic Interview Paradox: The Base Rate Fallacy
**Problem:** A rare disease affects $0.1\%$ ($0.001$) of the population. A lab test has a **$99\%$ True Positive Rate (Sensitivity)** and a **$5\%$ False Positive Rate**. A randomly selected patient tests positive. What is the probability they actually have the disease?

```
P(Disease) = 0.001       ===> P(Healthy) = 0.999
P(Pos | Disease) = 0.99  ===> P(Pos | Healthy) = 0.05

P(Pos) = (0.99 * 0.001) + (0.05 * 0.999) 
       = 0.00099 + 0.04995 = 0.05094

P(Disease | Pos) = (0.99 * 0.001) / 0.05094 ≈ 0.0194 (Only 1.94%!)
```
*Business Takeaway:* Even with a 99% accurate test, an alarm is false over 98% of the time when the underlying base rate is extremely rare! This is foundational for Fraud Detection and Intrusion Alert systems.

---

# Part 3: Common Probability Distributions

### 3.1 Discrete Distributions

| Distribution | Parameters | Support ($k$) | PMF $P(X=k)$ | Data Science Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Bernoulli** | $p$ (success prob) | $\{0, 1\}$ | $p^k(1-p)^{1-k}$ | Single user click or conversion ($0$ or $1$). |
| **Binomial** | $n$ trials, $p$ | $\{0, 1, \dots, n\}$ | $\binom{n}{k} p^k(1-p)^{n-k}$ | Number of conversions out of $n$ landing page visitors. |
| **Poisson** | $\lambda$ (rate per interval) | $\{0, 1, 2, \dots\}$ | $\frac{\lambda^k e^{-\lambda}}{k!}$ | Number of API requests per minute; server errors per hour. |

*Poisson Property:* The mean and variance of a Poisson distribution are identical:
$$\mathbb{E}[X] = \text{Var}(X) = \lambda$$

---

### 3.2 Continuous Distributions

#### 1. Normal (Gaussian) Distribution: $\mathcal{N}(\mu, \sigma^2)$
- Bell-shaped, symmetric about mean $\mu$.
- **Empirical 68-95-99.7 Rule:**
  - $68.27\%$ of observations fall within $\mu \pm 1\sigma$
  - $95.45\%$ fall within $\mu \pm 2\sigma$
  - $99.73\%$ fall within $\mu \pm 3\sigma$
- Standard Normal Distribution ($Z$): $\mu=0, \sigma=1$, transformed via $Z = \frac{x - \mu}{\sigma}$.

#### 2. Student's $t$-Distribution:
- Symmetric bell curve with **heavier tails** than the normal distribution.
- Governed by **degrees of freedom** ($\nu = n-1$). As $n \to \infty$, the $t$-distribution converges to the Standard Normal $\mathcal{N}(0, 1)$.
- **Crucial Rule:** Used instead of the $Z$-distribution whenever sample size is small ($n < 30$) AND the true population standard deviation $\sigma$ is unknown.

#### 3. Exponential Distribution:
- Models the **time elapsed between events** in a Poisson process with rate $\lambda$.
- **Memoryless Property:** $P(X > s + t \mid X > s) = P(X > t)$. The probability of waiting an additional $t$ minutes does not depend on how long you have already waited.

---

# Part 4: The Central Limit Theorem (CLT) & Sampling

### 4.1 Statement of the Central Limit Theorem
Regardless of the underlying population distribution (whether uniform, skewed, bimodal, or exponential), the **sampling distribution of the sample mean ($\bar{X}$)** approaches a Normal Distribution as the sample size $n$ becomes large ($n \ge 30$):
$$\bar{X} \sim \mathcal{N}\left(\mu, \frac{\sigma^2}{n}\right)$$

### 4.2 Standard Error of the Mean (SE)
$$\text{SE} = \frac{\sigma}{\sqrt{n}} \approx \frac{s}{\sqrt{n}}$$
The Standard Error measures the variability/dispersion of sample means around the true population mean. Notice that to cut the margin of error in half, **sample size must quadruple ($4\times$)** due to the square root in the denominator.

### 4.3 Confidence Intervals (CI)
A 95% Confidence Interval for the population mean:
$$\text{CI}_{95\%} = \bar{x} \pm t^* \cdot \frac{s}{\sqrt{n}}$$
- **Correct Interpretation:** *"If we repeated this identical sampling experiment 100 times and calculated confidence intervals, approximately 95 of those intervals would contain the true, fixed population mean $\mu$."*
- **Incorrect (Interview Trap):** "There is a 95% probability that the true mean lies between $A$ and $B$." (The population mean is a fixed constant, not a random variable; the *interval* is the random entity).

---

# Part 5: Hypothesis Testing & A/B Testing Framework

```
                          REALITY: H0 is TRUE       REALITY: H0 is FALSE
DECISION: Reject H0       TYPE I ERROR (α)           CORRECT (Power: 1 - β)
                          (False Alarm)              (True Discovery)

DECISION: Fail to Reject  CORRECT                    TYPE II ERROR (β)
          H0              (Confidence: 1 - α)        (Missed Opportunity)
```

### 5.1 The 5-Step Hypothesis Testing Workflow
1. **Formulate Hypotheses:**
   - Null Hypothesis ($H_0$): Status quo / no effect ($\mu_A = \mu_B$).
   - Alternative Hypothesis ($H_1$): Effect exists ($\mu_B > \mu_A$ or $\mu_A \neq \mu_B$).
2. **Select Significance Level ($\alpha$):** Typically $\alpha = 0.05$ (5% tolerance for Type I error).
3. **Compute Test Statistic:**
   - Two-sample independent $t$-test: $t = \frac{\bar{x}_B - \bar{x}_A}{\sqrt{\frac{s_A^2}{n_A} + \frac{s_B^2}{n_B}}}$
4. **Calculate $p$-value:** The probability of obtaining a test statistic at least as extreme as observed, assuming the null hypothesis is true.
5. **Make Business Decision:** If $p \le \alpha$, reject $H_0$; if $p > \alpha$, fail to reject $H_0$.

---

### 5.2 Common Misconceptions About $p$-values
- **What a $p$-value IS:** $P(\text{Data as extreme as observed} \mid H_0 \text{ is true})$.
- **What a $p$-value IS NOT:**
  - It is **NOT** the probability that the null hypothesis is true ($P(H_0 \mid \text{Data})$).
  - It does **NOT** measure the magnitude or practical business importance of an effect. A tiny, meaningless $0.01\%$ increase can be "statistically significant" ($p < 0.001$) if sample size $N = 10,000,000$.

---

# Part 6: Correlation, Covariance & Causation

### 6.1 Covariance vs. Pearson Correlation
- **Covariance:** $\text{Cov}(X, Y) = \frac{1}{n-1}\sum_{i=1}^n (x_i - \bar{x})(y_i - \bar{y})$.
  - Measures directional relationship, but its value is unbounded and depends on scale/units.
- **Pearson Correlation Coefficient ($r$):**
  $$r = \frac{\text{Cov}(X, Y)}{s_X \cdot s_Y} \quad (-1 \le r \le +1)$$
  - Standardized covariance. Measures the strength and direction of a strictly **linear** relationship.

### 6.2 Spearman Rank Correlation ($\rho$)
Computes Pearson's $r$ over the **ranks** of the data rather than raw values:
- Measures **monotonic relationships** (whether variables increase together, regardless of linearity).
- Immune to extreme outliers.

---

### 6.3 Confounding & Simpson's Paradox
**Simpson's Paradox:** A statistical trend that appears in different groups disappears or reverses when the groups are aggregated.

*Classic Example (UC Berkeley Admissions):*
- Overall: Men had higher admission rates than women.
- Sub-department breakdown: In nearly every single department, women had equal or higher admission rates than men!
- **Confounding Variable:** Women applied disproportionately to competitive departments with very low acceptance rates, while men applied to less competitive departments with high overall acceptance.

---

# Part 7: Required Deliverable: Executive Translation Brief

### Context
A product team tested a "One-Click Checkout" button ($B$) against the standard multi-step checkout ($A$).
- Sample size: $n_A = 50,000, n_B = 50,000$
- Conversion rate: $CR_A = 3.20\%$, $CR_B = 3.65\%$
- Relative lift: $+14.06\%$
- $p$-value: $0.00012$
- $95\%$ Confidence Interval of Lift: $[+7.1\%, +21.0\%]$

---

### Executive Brief Document

```markdown
# EXECUTIVE DECISION BRIEF: One-Click Checkout Experiment

**Date:** October 2026  
**To:** Chief Product Officer & VP of E-Commerce  
**From:** Senior Data Science Engineer  
**Recommendation:** **FULL ROLLOUT (Ship Variant B to 100% of Traffic)**

---

### 1. Executive Summary
Over a two-week controlled experiment covering 100,000 customer sessions, introducing the "One-Click Checkout" experience produced a **statistically confirmed +14% increase in completed purchases**. 

Based on current annual GMV, this conversion improvement represents an estimated **$2.4M to $4.1M in incremental annualized revenue**.

---

### 2. Plain-Language Experiment Findings

| Metric | Current Experience (Control) | New One-Click Checkout | Plain-English Business Impact |
| :--- | :--- | :--- | :--- |
| **Completed Orders per 1,000 Visitors** | 32 orders | **36.5 orders** | **+4.5 extra paying customers** per 1,000 visitors |
| **Statistical Certainty** | N/A | **99.98% confidence** | The odds of this result being random noise are under 1 in 8,000. |
| **Projected Annual Impact** | Baseline ($0) | **+$3.2M / year** | 95% confident the permanent gain is at least +$1.8M/yr. |

---

### 3. Risk & Guardrail Assessment
- **Order Cancellation Rate:** Monitored daily as a guardrail metric. Cancellation rate remained flat ($1.1\%$ vs $1.2\%$), confirming that the frictionless checkout did not trigger accidental orders.
- **Mobile vs. Desktop:** Gains were strongest on mobile (+19% lift), successfully solving our checkout drop-off bottleneck on mobile devices.

### 4. Recommended Action Plan
1. Ramp Variant B from 50% to 100% across all traffic starting Monday at 02:00 UTC.
2. Retire legacy multi-step checkout code to eliminate maintenance overhead.
```

---

# Part 8: Structured Interview Question Bank

### Q1: What is the difference between Type I and Type II errors? Which is worse in fraud detection?
**Answer:**
- **Type I Error ($\alpha$, False Positive):** Rejecting the null hypothesis when it is true. (e.g., Flagging a legitimate customer transaction as fraud).
- **Type II Error ($\beta$, False Negative):** Failing to reject the null hypothesis when an effect exists. (e.g., Letting an actual fraudulent transaction slip through unnoticed).
- In high-value fraud detection, a **Type II error is typically much worse** because an undetected fraudster causes direct financial loss, chargebacks, and legal liability, whereas a Type I error merely causes temporary user friction (e.g., requiring an SMS OTP).

### Q2: What happens if an A/B test is evaluated early and stopped as soon as $p < 0.05$ ("Peeking Problem")?
**Answer:**
Continuously checking results and stopping an A/B test the moment $p < 0.05$ causes severe **$\alpha$-inflation (False Positive inflation)**. Because random fluctuations will inevitably cross the $0.05$ threshold by chance at some point during the experiment, the true false positive rate spikes from $5\%$ to $30-40\%$. To prevent this, teams must either use fixed-horizon sample sizes or implement **Sequential Testing** (e.g., Wald's Sequential Probability Ratio Test or alpha-spending functions).

### Q3: Explain why Pearson correlation cannot detect a perfect parabolic relationship ($Y = X^2$).
**Answer:**
Pearson correlation strictly measures the strength and direction of a **linear** association. For $Y = X^2$ centered around $0$ (e.g., $X \in [-3, -2, -1, 0, 1, 2, 3]$), positive deviations on the right cancel out negative deviations on the left in the covariance formula, resulting in $\text{Cov}(X, Y) = 0$ and $r = 0$. Spearman rank correlation also fails here because the relationship is non-monotonic.

---

# Part 9: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   STATISTICS & PROBABILITY CHEAT SHEET                 │
├────────────────────────────────────────────────────────────────────────┤
│ 1. CENTRAL TENDENCY:                                                   │
│    - Right Skew: Mean > Median > Mode (Outliers pull mean right).      │
│    - Left Skew : Mean < Median < Mode.                                 │
│    - Bessel's Correction: Divide by n-1 for unbiased sample variance.  │
├────────────────────────────────────────────────────────────────────────┤
│ 2. PROBABILITY & BAYES:                                                │
│    - P(A|B) = [P(B|A) * P(A)] / P(B)                                   │
│    - False Positive Paradox: Low base rate means most alerts are false.│
├────────────────────────────────────────────────────────────────────────┤
│ 3. DISTRIBUTIONS:                                                      │
│    - Binomial: n independent trials with prob p.                       │
│    - Poisson: Events in fixed interval (Mean = Variance = λ).          │
│    - Normal: 68-95-99.7 rule. CLT guarantees sample mean is normal.   │
│    - t-distribution: Small sample (n < 30) or unknown σ. Heavier tails.│
├────────────────────────────────────────────────────────────────────────┤
│ 4. HYPOTHESIS TESTING:                                                 │
│    - Type I Error (α): False Positive (Reject true H0).                │
│    - Type II Error (β): False Negative (Fail to reject false H0).      │
│    - Power (1 - β): Probability of detecting a real effect.            │
│    - p-value: P(observed data | H0 is true). NOT P(H0 is true)!        │
├────────────────────────────────────────────────────────────────────────┤
│ 5. CORRELATION:                                                        │
│    - Pearson r: Linear (-1 to +1). Sensitive to outliers.              │
│    - Spearman ρ: Monotonic. Uses ranks. Robust to outliers.            │
│    - Simpson's Paradox: Aggregate trend reverses when subgrouped.      │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 10: 100-Mark Self-Assessment Examination

### Section A: Conceptual & Formulas (25 Marks — 5 marks each)

**Q1:** What is the relationship between the Mean and the Median in a positively (right) skewed distribution?  
(a) Mean = Median  
(b) Mean < Median  
(c) Mean > Median  
(d) Variance = Mean  

**Q2:** If the sample size $n$ is increased from 100 to 400, what happens to the Standard Error of the Mean?  
(a) Doubled ($2\times$)  
(b) Halved ($\frac{1}{2}\times$)  
(c) Reduced by a factor of 4  
(d) Remains unchanged  

**Q3:** What is the variance of a Poisson distribution with parameter $\lambda = 9$?  
(a) 3  
(b) 9  
(c) 81  
(d) 18  

**Q4:** A hypothesis test yields a $p$-value of $0.03$. What does this indicate at a significance level of $\alpha = 0.05$?  
(a) There is a 3% chance the null hypothesis is true.  
(b) Reject the null hypothesis; the observed result is statistically significant.  
(c) Fail to reject the null hypothesis.  
(d) Accept the alternative hypothesis with 97% certainty.  

**Q5:** In a medical screening scenario with high sensitivity (99%) and a very rare disease prevalence (1 in 10,000), why is a positive test result usually a false alarm?  
(a) The test sensitivity is too low.  
(b) The Base Rate Fallacy: The absolute number of false positives from the large healthy population dwarfs the true positives.  
(c) Type II error inflation.  
(d) Simpson's Paradox.  

---

### Section B: Calculation & Probability Problems (25 Marks)

**Q6 (10 Marks):** Given the dataset: `[4, 8, 12, 16, 20]`:
1. Calculate the sample mean $\bar{x}$.
2. Calculate the sample standard deviation $s$ (using Bessel's $n-1$ correction).

**Q7 (15 Marks):** In an enterprise data center, the number of hardware server failures follows a Poisson distribution with an average of $\lambda = 2$ failures per day.
1. Write the formula for $P(X = k)$.
2. Calculate the exact probability that on a given day, exactly 0 server failures occur ($e^{-2} \approx 0.1353$).
3. Calculate the probability of experiencing at least 1 failure.

---

### Section C: A/B Testing & Inferential Scenarios (30 Marks)

**Q8 (15 Marks):** An e-commerce analyst runs an A/B test with 20 different button color variations simultaneously against a control, without applying any multiple testing corrections. If each test uses $\alpha = 0.05$, calculate the family-wise error rate (the probability of getting at least one false positive result by pure chance). How does the **Bonferroni correction** resolve this?

**Q9 (15 Marks):** Explain how Simpson's Paradox could lead a data science team to falsely conclude that a new recommendation algorithm performed worse than the old algorithm, when in fact it was superior in every customer demographic.

---

### Section D: Real-World Business Translation (20 Marks)

**Q10 (20 Marks):** Your machine learning model predicts loan default risk for a commercial bank. The model has an AUC-ROC of $0.88$, a Precision of $35\%$, and a Recall of $82\%$.
The VP of Risk says: *"Why is precision only 35%? That means 65% of the loans flagged for denial are actually safe! This model is useless."*
Write a persuasive, technically sound response explaining why a 35% precision is highly valuable given the cost asymmetry of loan defaults vs. lost interest.

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(c)** Mean > Median (The extreme high values pull the mean upward).
- **Q2:** **(b)** Halved. $\text{SE} = \frac{s}{\sqrt{400}} = \frac{s}{20}$, which is half of $\frac{s}{10}$.
- **Q3:** **(b)** 9. In a Poisson distribution, $\text{Var}(X) = \lambda = 9$.
- **Q4:** **(b)** Reject $H_0$ because $p = 0.03 \le \alpha = 0.05$.
- **Q5:** **(b)** The Base Rate Fallacy.

### Section B Solutions
**Q6 Solution:**
1. Mean: $\bar{x} = \frac{4 + 8 + 12 + 16 + 20}{5} = \frac{60}{5} = 12$.
2. Deviations squared: $(4-12)^2 = 64, (8-12)^2 = 16, (12-12)^2 = 0, (16-12)^2 = 16, (20-12)^2 = 64$.
   - Sum of squared deviations $= 64 + 16 + 0 + 16 + 64 = 160$.
   - Sample Variance: $s^2 = \frac{160}{5-1} = \frac{160}{4} = 40$.
   - Sample Standard Deviation: $s = \sqrt{40} \approx 6.32$.

**Q7 Solution:**
1. $P(X = k) = \frac{\lambda^k e^{-\lambda}}{k!}$.
2. For $k = 0$: $P(X = 0) = \frac{2^0 e^{-2}}{0!} = \frac{1 \cdot 0.1353}{1} = 0.1353$ (13.53%).
3. At least 1 failure: $P(X \ge 1) = 1 - P(X = 0) = 1 - 0.1353 = 0.8647$ (86.47%).

### Section C Solutions
**Q8 Solution:**
- Probability of no false positive in 1 test: $1 - 0.05 = 0.95$.
- Probability of no false positive across 20 independent tests: $(0.95)^{20} \approx 0.3585$.
- **Family-Wise Error Rate (FWER):** $1 - 0.3585 = 0.6415$ (**64.15% chance of at least one false discovery!**).
- **Bonferroni Correction:** Divide the target alpha by the number of comparisons: $\alpha_{\text{adjusted}} = \frac{0.05}{20} = 0.0025$. Each individual variation must reach $p \le 0.0025$ to be declared significant.

**Q9 Solution:**
Simpson's Paradox occurs when a confounding variable divides the dataset into unequal segments. If the new algorithm was rolled out during a seasonal surge predominantly to new, first-time users (who have an inherently lower baseline conversion rate of 1%) while the old algorithm served loyal power users (baseline 10%), the aggregated conversion rate of the new algorithm appears lower, even if it boosted first-time users from 1% to 2% and loyal users from 10% to 12%.

### Section D Solutions
**Q10 Solution:**
*"A commercial loan default results in an average catastrophic loss of $50,000 in unpaid principal, whereas approving a good loan yields $3,000 in net interest margin. The cost of a False Negative ($50,000) is nearly $17\times$ higher than the cost of a False Positive ($3,000). At 82% recall, our model successfully captures over 8 out of 10 catastrophic defaults. A 35% precision means that for every 10 flagged loans, 3.5 are true defaults ($175,000 in capital saved) at the cost of requiring manual credit reviews on 6.5 applications. The net expected economic value saved far exceeds the cost of credit underwriting friction."*
