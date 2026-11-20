# Q1: Where are Dubai prices rising fastest?

> Fill in every `[ ]` from your own results in `databricks/04_q1_price_hotspots.ipynb`. Round sensibly (whole AED, one decimal for %). Delete these instruction lines when you're done.

**Scope:** residential sales (apartments and villas/townhouses) registered with the DLD, 2015 to October 2026. Price measured as the median AED per sq ft, excluding price outliers. A community × segment × bedroom band is only compared when it has at least **50 sales in each year** being compared.

---

## Headline

> One sentence a recruiter remembers. Combine the breadth result (Finding 1) with the 2026 momentum result (Finding 4).

[ ]

---

## Finding 1: The 2025 rise was broad, not just a few hotspots

- In 2025, **[ ]%** of the [ ] qualifying apartment groups saw prices rise, with a median change of **[ ]%** year on year.
- For villas and townhouses: **[ ]%** rising, median **[ ]%**.
- For comparison, in 2019 only [ ]% of apartment groups were rising (median [ ]%).

**So what:** [ ]

---

## Finding 2: The fastest risers, and why the headline numbers need care

- On the raw numbers, the top apartment riser was **[community, bedrooms]**, up **[ ]%** (AED [ ] → [ ] per sq ft).
- But its off-plan share moved from [ ]% to [ ]% of sales. Off-plan units sell at about [ ]× the price per sq ft of ready units there. Within ready homes alone, the median rose about [ ]%.
- Excluding groups whose off-plan share moved by 25+ points ([ ] of [ ] groups), the top risers were:
  1. [ ] ([ ]%)
  2. [ ] ([ ]%)
  3. [ ] ([ ]%)

**So what:** [ ]

---

## Finding 3: Growth across the cycle, 2021 to 2025

- [ ] groups had 50+ sales in both 2021 and 2025.
- The highest compound annual growth was **[ ]** at **[ ]% a year** ([ ]% in total).
- The busiest group, **[ ]**, grew at [ ]% a year.

**So what:** [ ]

---

## Finding 4: 2026, prices holding while activity cools

- Comparing January–October 2026 with January–October 2025 (like for like), **[ ]** of [ ] qualifying apartment groups are **still rising**, and **[ ]** have **turned down**.
- The largest groups that turned down: [ ], [ ], [ ].

**So what:** [ ]

---

## Method and limitations

- **Median, not average:** robust to unusual sales.
- **Outliers:** sales below the 0.5th or above the 99.5th percentile of AED per sq ft for their property type are excluded.
- **Minimum sample:** 50 sales per year per group. This removes [ ]% of 2025 groups but only [ ]% of 2025 sales.
- **Mix effects:** a change in median price per sq ft can reflect a change in *which* homes sold (e.g. new off-plan launches), not only a change in value. Groups whose off-plan share moved by 25+ points are flagged.
- **No repeat-sales index:** the data has no unit ID, so the same unit can't be tracked across sales.
- **Price per sq ft ignores quality, age, view and floor.**
- **Source:** Dubai Land Department open data (Dubai Data portal), extract of 6 Nov 2026. Independent analysis, not official statistics.
