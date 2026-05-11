# Artea Coupon Campaign: A/B Test & Causal Targeting Analysis

> A causal inference project that figures out whether discount coupons actually drive incremental revenue — and which customers actually deserve them.

**Course:** BANA 277 · Customer & Social Analytics (UC Irvine, MSBA)
**Team 12:** Jaya Sruthi Perikala · Angie Pang · Andrea Pan · Carson Pimental
**Tools:** Python (`pandas`, `statsmodels`, `scipy`, `matplotlib`)

---

## 📌 Project Purpose

Artea is an online retailer that sells handmade clothing and accessories. On the surface, things look healthy — visitors spend real time on the site, referral rates are strong, and traffic keeps growing. But hidden behind those numbers is one ugly statistic: **87% of website visitors never make a purchase**.

To close that gap, the CEO considered sending 20% discount coupons to recent visitors. She had two real concerns about doing it:

- Discounts can "train" customers to wait for promotions, which erodes margins long-term
- Some customers would have bought anyway, so coupons sent to them are just giving money away

This project answers four questions the team needed to make a decision:

1. Did the coupon actually increase transactions and revenue?
2. Which customers should be targeted in the next campaign?
3. How many transactions and how much revenue should Artea expect from that targeting?
4. Should demographic data (gender, minority status) change the targeting strategy — and is it even ethical to use?

The point isn't just to measure averages. It's to find the customers for whom the coupon *causes* incremental purchases, not the ones who would have bought regardless.

---

## 📊 Dataset Description

The analysis uses two main datasets from a randomized A/B experiment, plus a demographic add-on supplied by a third-party data vendor (Trackify).

### `AB_Test` — 5,000 users (the experiment)

Users who had visited the website in the past two months but hadn't purchased. Half were randomly assigned a 20% off coupon, half got nothing.

| Variable | Description |
|---|---|
| `test_coupon` | 1 = received coupon, 0 = control |
| `trans_after` | Transactions made in the month after the experiment |
| `revenue_after` | Total revenue after the experiment (USD) |
| `shopping_cart` | Added to cart in last visit but didn't buy (1/0) |
| `num_past_purch` | Number of previous purchases |
| `spent_last_purchase` | USD spent on previous purchase |
| `weeks_since_visit` | Weeks since last website visit |
| `browsing_minutes` | Minutes on site during last visit |
| `channel_acq` | 1=Google · 2=Facebook · 3=Instagram · 4=Referral · 5=Other |

### `Next_Campaign` — 6,000 users (the future pool)

Same features as above (minus the experiment outcome and treatment columns). This is the pool we apply our targeting rule to.

### Demographic add-on (Part B)

A third-party vendor supplied predicted demographic attributes for both datasets:

| Variable | Description |
|---|---|
| `minority` | Predicted minority group membership (1/0) |
| `non_male` | Predicted non-male gender (1/0) |

These are predicted by an algorithm, not self-reported — an important caveat for both accuracy and ethics.

---

## 🏷 Methodology

Averages alone don't answer "who should we target," so the analysis layers a regression-driven uplift workflow on top of the basic A/B test.

### Step 1 — Validate the experiment

Started with group means and two-sample t-tests on `trans_after` and `revenue_after` to check whether the coupon had any overall effect.

### Step 2 — Find heterogeneous effects with interaction terms

Ran an OLS regression of revenue on customer features **plus interaction terms** between `test_coupon` and each feature:

```text
revenue_after ~ test_coupon + shopping_cart + weeks_since_visit
              + browsing_minutes + num_past_purch + spent_last_purchase
              + C(channel_acq)
              + test_coupon:shopping_cart + test_coupon:weeks_since_visit
              + test_coupon:browsing_minutes + test_coupon:num_past_purch
              + test_coupon:spent_last_purchase + test_coupon:C(channel_acq)
```

The interaction terms tell us *which customer traits amplify or shrink the coupon's effect* — this is the heart of the targeting question.

### Step 3 — Compute uplift by segment

For each segment defined by a significant interaction (cart status, purchase history bin, acquisition channel), computed:

> **Uplift = Avg(Treatment) − Avg(Control)** — calculated for both transactions and revenue.

### Step 4 — Pick segments that win on both metrics

A segment only made the cut if the coupon increased transactions *and* didn't burn revenue.

### Step 5 — Apply the rule to `Next_Campaign`

Identified which of the 6,000 future-campaign users met the targeting criteria, then estimated incremental transactions and revenue using the uplift values from the AB test.

### Step 6 — Test demographics separately

Re-ran the regression with `test_coupon:minority` and `test_coupon:non_male` interaction terms added, to check whether the targeting strategy should change based on demographic group.

---

## 🔍 Key Findings & Insights

### Finding 1 — The blanket coupon backfires on revenue

| Metric | Control | Coupon | Difference | p-value |
|---|---|---|---|---|
| Avg transactions | 0.126 | 0.152 | **+0.026 (+20.8%)** | 0.027 ✅ |
| Avg revenue | $7.78 | $7.54 | **−$0.24** | 0.718 ❌ |

The coupon clearly drove **more purchases** — that part is real. But the 20% discount ate into the average ticket size, so revenue actually fell slightly (and the difference isn't statistically significant in either direction). Sending coupons to everyone is a losing trade.

### Finding 2 — Three interactions reveal where the coupon actually pays off

From the OLS regression:

| Interaction | Coef. | p-value | What it means |
|---|---|---|---|
| `test_coupon × shopping_cart` | +7.69 | 0.021 | Cart-abandoners convert when nudged |
| `test_coupon × num_past_purch` | −1.27 | 0.000 | Loyal customers would buy anyway — discount just destroys margin |
| `test_coupon × Instagram channel` | +3.11 | 0.033 | Instagram-acquired users are more promotion-responsive |

### Finding 3 — The targeting rule

> **Send coupons only to users who: added something to cart, have 0–2 past purchases, AND were acquired through Facebook, Instagram, or Referral.**

### Finding 4 — Expected impact on the next campaign (6,000 users)

| Metric | Value |
|---|---|
| Customers targeted | 668 (11.13% of pool) |
| Incremental transactions | ~51 |
| Incremental revenue | **+$922.51** |
| Total expected revenue | $9,704.70 |

Compared to blasting the whole list, targeted sending flips a *revenue loss* into a *revenue gain* while still capturing the transaction lift.

### Finding 5 — Demographics don't change the answer

Adding `minority` and `non_male` interactions to the model:

| Interaction | p-value | Conclusion |
|---|---|---|
| `test_coupon × minority` | 0.639 | Not significant |
| `test_coupon × non_male` | 0.164 | Not significant |

The coupon doesn't work meaningfully differently across demographic groups. Targeting based on gender or minority status would add legal and ethical risk without improving results — there's no analytical reason to do it.

One interesting side note though: Google-acquired users skew much more toward minority customers than Facebook or Instagram do. So a behavior-based rule that excludes Google could still create disparate impact, even though it never touches a demographic variable directly. Worth keeping an eye on.

---

## 📊 Visualizations

All figures are generated in `Team12_Artea.ipynb`.

| # | Figure | What it shows |
|---|---|---|
| 1 | Total & average transactions by group | Coupon group has +66 transactions — lift is real |
| 2 | Total & average revenue by group | Revenue is flat-to-down despite more transactions |
| 3 | Revenue uplift by segment | Cart-adders (+$1.38) and 1–2 purchase buyers (+$1.28) lead the pack |
| 4 | Avg revenue & transactions by minority / gender | Visual check on demographic differences |
| 5 | Significance of demographic interaction terms | Both p-values well above 0.05 |
| 6 | Minority distribution across acquisition channels | Google brings in the most diverse audience |

---

## 🚀 Business Implications

### What Artea should actually do

- **Don't blanket discount.** The A/B test makes it clear: a universal 20% coupon increases purchases but loses revenue. Selective targeting is non-negotiable.
- **Use behavioral signals, not who the customer is.** Shopping cart activity, recent purchase count, and acquisition channel are all you need to know.
- **Skip the demographic data purchase.** The vendor's data doesn't improve targeting accuracy, and using it opens up disparate impact risk that isn't worth the cost.
- **Keep experimenting.** This rule comes from one experiment. Customer behavior shifts, channels evolve, and uplift estimates decay. Treat the targeting rule as something to re-test, not set-and-forget.

### Risks to monitor

| Risk | Level | Mitigation |
|---|---|---|
| Generalization of A/B results | High | Re-run experiments quarterly |
| Promotion dependency | Med | Cap coupon frequency per customer |
| Fairness / disparate impact | Low–Med | Audit channel-based rules for demographic skew |

### Bigger-picture takeaway

The most valuable lesson from this project isn't the specific rule. It's the discipline of asking *who the coupon actually changed behavior for* — instead of celebrating an average lift that hides a money-losing reality underneath. Good targeting starts with causal effects, not correlations.

---

## 📁 Repository Structure

```
artea-coupon-targeting/
├── README.md                          # This file
├── notebooks/
│   └── Team12_Artea.ipynb             # Full analysis code
├── data/
│   ├── Artea_data.xlsx                # AB_test + Next_Campaign
│   └── Artea_(B)_data.xlsx            # With demographic columns
├── report/
│   └── Team12_Artea_Paper.pdf         # Written report
├── slides/
│   └── Team12_Artea_Slides.pdf        # Presentation deck
└── figures/                           # Exported charts
```

---

*Case materials adapted from Ascarza & Israeli, "Artea: Designing Targeting Strategies" (HBS 9-521-021) and "Artea (B): Including Customer-level Demographic Data" (HBS 9-521-022). Artea, Alex Campbel, and Trackify are fictional.*
