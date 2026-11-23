# Detecting Cross-Sectional Mispricing in Fungible Consumer Goods on eBay

Project originally done January 2026; code posted November 2026. [LinkedIn post](https://www.linkedin.com/posts/andrew-kangmin-kim_andrews-weekly-analysis-week7designing-share-7423217944475897857-T85q/)

## Question

Can economically meaningful mispricing be detected among *AirPods Pro 2* Buy-It-Now listings on eBay US by:

- clustering identical items with hard, interpretable rules,
- adjusting prices for seller (counterparty) risk and item-level quality signals,
- and flagging only large residual deviations from a robust fair price?

This is a stat-arb–style residual pricing problem in a two-sided marketplace, not risk-free arbitrage.
Execution is paper trading only.

| Finance concept | Marketplace counterpart |
|---|---|
| Underlying asset | AirPods Pro 2 |
| Counterparty / credit risk | Seller credibility (feedback %, feedback count, Top Rated status) |
| Idiosyncratic risk | Defects and missing items stated in the listing text |
| Mispricing signal | Residual price deviation within a cluster |

## Pipeline

1. **Ingest:** paginated pull of Buy-It-Now listings from the official eBay Browse API.
2. **Normalise:** lower-case the listing titles and strip URLs and symbols.
3. **Extract identity:** use rules to read model, case type (USB-C / Lightning), condition and box contents from the title.
4. **Hard filters:** keep confirmed AirPods Pro 2 listings only; drop replicas, parts-only listings and prices below $70.
5. **Quality flags:** cosmetic damage, heavy use, missing accessories, engraving.
6. **Seller risk:** an additive score from feedback percentage, feedback count and Top Rated status.
7. **Cluster:** group listings by case type × condition; clusters with fewer than 3 listings roll up into a parent cluster.
8. **Fair price:** a robust (Huber) log-linear regression within each cluster, with features centred on the cluster mean.
9. **Signal:** the price residual, standardised within its cluster (`arb_score`).
10. **Gate and threshold:** keep only sellers with a risk score ≤ 2, then flag `arb_score ≤ −1.0` as buy signals.

## Results (single snapshot)

| Stage | Listings |
|---|---|
| Pulled from the Browse API (capped at 50 pages × 50) | 2,500 |
| After hard filters | 1,238 |
| Scored (clusters large enough to model) | 1,234 |
| Passing the seller-risk gate | 355 |
| **Buy signals** (`arb_score ≤ −1.0`) | **29** |

- **Clusters:** most listings fall in "case unknown, condition unknown" (902) and "USB-C, condition unknown" (180),
  because titles rarely state condition. Seven clusters had enough listings for their own price anchor.
- **Anchor prices:** the two largest clusters both anchor at a median of $115 (MAD ≈ $15).
- **The signal is skewed:** scores range from −1.5 to +12.7. Overpricing is far more extreme than underpricing,
  so genuine discounts are modest and the useful signal lies in the left tail.
- **Seller risk dominates:** of the 1,238 filtered listings, 1,171 come from sellers who are not Top Rated.

Raw listing data is not included: listings, item IDs and seller usernames come from eBay and are not
redistributed. The notebook's outputs have been cleared for the same reason.

## Running it

1. Create an application keyset in the [eBay Developers Program](https://developer.ebay.com/).
2. Provide the credentials as environment variables, or as Colab secrets with the same names:
   ```bash
   export EBAY_CLIENT_ID=...
   export EBAY_CLIENT_SECRET=...
   ```
3. Install and run:
   ```bash
   pip install -r requirements.txt
   jupyter notebook ebay_mispricing.ipynb
   ```

Listings change continually, so each run analyses a new snapshot of the market.

## Limitations

- Identity and quality come from titles only; item descriptions and images are not used.
- A single cross-sectional snapshot, capped at the first 2,500 search results, with no follow-up on
  whether flagged listings were really mispriced.
- Fees, shipping costs, resale prices and the probability of a successful resale are not modelled.

## License

[MIT](LICENSE)
