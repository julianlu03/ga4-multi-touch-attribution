# Multi-Touch Attribution Comparison: Last-Touch vs. Linear vs. Time-Decay

*Dataset: `bigquery-public-data.ga4_obfuscated_sample_ecommerce` (BigQuery, Google Merchandise Store) · Tools: BigQuery SQL, Python (Pandas)*

[Tableau Dashboard →](#) *(placeholder link)*

---

## Problem

Don't blindly trust last-click. If you're only giving credit to the channel or source your customer came from right before purchasing, you might be skipping much of the journey it took to get them there in the first place. In this project, I challenge that assumption: I extracted three months of Google Merchandise Store data from BigQuery and built three different attribution models to see how each one represented channel contribution.

---

## Key Decisions

### Is It Worth Building?

Before doing anything else, I had to decide whether building multiple attribution models was even worthwhile. If the large majority of conversions in this three-month period were single-touch (only one session before conversion), the results of every model would look basically identical to Last-Touch. Before extracting the full dataset, I queried session counts per purchaser directly in BigQuery to build a histogram of conversions against number of pre-purchase sessions. With 2,084 (47%) single-session purchasers and 2,335 (53%) multi-session purchasers, this confirmed there was enough multi-touch behavior in the data to make attribution modeling meaningful.

### First-Purchase Scoping

The original dataset (`bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`) is event-level: every row is a single event (e.g. clicking a product, adding to cart), with each event tied to a specific user, session, and the channel/source associated with that session. For attribution modeling, I only cared about session-level data — how many sessions it took to convert, and the source/channel tied to each one.

I filtered the data to sessions up to and including each user's first purchase (behavior after that point can be explained by other factors, like prior brand experience or remarketing), initially using `event_name = "session_start"` to identify each session. However, `session_start` turned out to be inconsistent — some sessions never generated one at all, causing 62 purchasers (~1.4% of the population) to disappear entirely from the initial extraction. The fix: rank each session's events by timestamp and anchor to the first event of that session, whatever type it happens to be, instead of requiring `session_start` specifically. This resolved the missing-session issue completely.

### Handling Missing Source/Channel Data

After building the first two models (Last-Touch as a baseline, and Linear), I noticed the single most-credited channel was `Unknown / redacted` — around 36% of credit under both models. I had combined the `source` and `medium` fields into one `channel` field (e.g. source = google, medium = organic → channel = `google / organic`), where if *either* field held an unattributable value (`(data deleted)` or `<Other>`), the whole channel collapsed to `Unknown / redacted`.

After finding how much credit that bucket was absorbing, I switched to a less strict rule: only collapse to `Unknown / redacted` if *both* source and medium are unattributable. Otherwise, keep whatever information is available (e.g. source = `<Other>`, medium = organic → channel = `Unknown source / organic`). This kept as much real information as possible without letting the channel field become misleading.

### Choosing the Half-Life

After Last-Touch and Linear, I built a Time-Decay model. Time-decay discounts each touchpoint's credit based on how much time passed between that touchpoint and the actual conversion. Selecting the half-life (the number of days until a touchpoint's credit decays to 50% of its original value) is an important decision.

**The math:**
```
weight = 0.5 ^ (days_before / half_life)
```
Add up the raw weight for every touchpoint in a path, then divide each one by that total (normalization) to get the final credit weights.

Example, half-life = 7:
```
0.5 ^ (10/7) = 0.372          0.5 ^ (0/7) = 1
Normalized:  0.372/1.372 = 27.1%     1/1.372 = 72.9%
```

Shorter half-lives punish touchpoints further from conversion and behave more like Last-Touch. Longer half-lives favor earlier touchpoints and behave more like Linear.

Looking only at multi-touch purchasers, the number of days between a user's first touchpoint and their conversion has a mean of 10.21 days, a median (50th percentile) of 3.83 days, and a 75th percentile of 14.26 days — a distribution heavily skewed right by a long tail of slower-converting journeys. I wanted to punish that long tail (touchpoints two-plus weeks out) with a shorter half-life, without fully discounting shorter, 3-5 day paths the way Last-Touch effectively does. I picked the commonly-cited industry-standard 7-day half-life as a reasonable middle ground for this distribution.

---

## Findings

**[Insert chart: Last-Touch vs. Linear vs. Time-Decay, full population]**

Comparing the three models, the Last-Touch model assigned noticeably less credit to organic search and more credit to referral traffic than the multi-touch models did. This traces back to where these channels typically sit in the funnel: organic search is more likely to be an early-funnel touchpoint, while referral traffic is systematically more likely to be the last touchpoint before purchase.

Filtering to multi-touch journeys only, this gap widened and became even more apparent.

**[Insert chart/numbers: model comparison, multi-touch only]**

---

## Scope & Limitations

Given how close Time-Decay landed to Linear, I also tested a shorter, 2-day half-life. Results barely moved despite dropping the half-life by more than 3x — `google / organic`, for example, went from 28.04% (half-life = 7) to 27.67% (half-life = 2), a difference of only 0.37 points. This is likely explained by the population structure of the data rather than the half-life choice itself.

Generally, model choice matters less here specifically because of that structure. With 47% of conversions being single-touch, nearly half the dataset has the same credit outcome no matter which model is applied: all credit goes to the one session that exists.

Separately, roughly 24.6% of last-touch credit (closer to 28.7% among multi-touch purchasers specifically) lands in `Unknown / redacted`, since GA4's obfuscation strips identifiable source/medium data from a meaningful share of sessions. No attribution model can credit a channel it can't identify — unattributable traffic is a real, structural limitation of this dataset, not something any of these three models can resolve.

---

## Recommendations

For an ecommerce retailer with a channel mix like the Google Merchandise Store's — heavy organic/direct traffic, meaningful referral presence — evaluating channel performance purely through last-touch conversion counts (the default in many out-of-the-box analytics reports) would systematically underweight organic search's contribution to the funnel and overweight referral's.

Organic search brings people in, but it rarely gets to be the channel that "wins" the conversion. Referral traffic, on the other hand, tends to arrive after the decision is largely made. A team at a business like this, making channel investment decisions off last-touch alone, would be biased toward deprioritizing early-funnel discovery channels in favor of channels that simply happen to close what was already likely to happen.

This isn't a case for blindly reallocating budget toward organic, however. This analysis measures how credit is *distributed* across models, not return on ad spend or cost per acquisition, so it can't support a dollar-for-dollar reallocation claim on its own.

What it does support is a more specific, actionable recommendation: before making a budget decision that affects organic or paid search investment, that decision should be informed by a multi-touch model (Linear or Time-Decay), not Last-Touch alone.

For a retailer with this purchase pattern — roughly half of customers converting in a single session — model choice is close to irrelevant for that half, since every model assigns the same credit when there's only one touchpoint to assign it to. The difference between models matters specifically for customers with longer, multi-touch consideration journeys. A business like this realistically shouldn't overhaul its entire attribution stack, since that's a lot of operational cost for a change that doesn't even affect half its customers. A more targeted approach: apply a multi-touch model selectively, to the segments or product categories where longer consideration paths are common.

Finally, this retailer sees roughly a quarter of touchpoints (closer to 28% among its longest, most complex journeys) carrying no identifiable channel. No attribution model can assign credit to a channel that isn't known. Improving data quality — UTM hygiene, consent-driven data loss, first-party tracking investment — would do more for attribution accuracy here than swapping models ever could, and would ultimately clarify which channels are actually worth the investment.
