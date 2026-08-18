# Competitor Gap Analysis

Three Google Colab notebooks that compare your site against a competitor's using embeddings and topic clustering, not keyword lists. Upload two crawl exports, run the cells, get back a topic map showing exactly where a competitor has more depth or breadth than you, sorted by how big the gap is.

No local setup, no Python environment to manage. Everything runs in the browser on Google's hardware. The `rising-tide.ipynb` notebook costs nothing to run; the other two spend a few cents to a few dollars on an LLM call, entirely under your control.

## Why clustering instead of keywords

A keyword list tells you what you're missing. It doesn't tell you whether a competitor has two pages on a topic or twenty, whether your gaps are structural or just tactical, or where you're already strong enough to defend. These notebooks treat content as structure: pages get embedded into vectors, grouped into topic clusters, and compared cluster by cluster. The full reasoning behind this is in [WALKTHROUGH.md](WALKTHROUGH.md).

## The three tiers

Each tier builds on the one before it. Start with Rising Tide; move up only if you need what the next tier adds.

| Notebook | Method | What it adds | Cost |
|---|---|---|---|
| [`rising-tide.ipynb`](rising-tide.ipynb) | Sentence Transformers + K-Means | Topic clusters, breadth/depth charts, a UMAP map | Free, fully local |
| [`open-water.ipynb`](open-water.ipynb) | BERTopic + capped LLM synthesis | Auto-labeled clusters, a prioritized gap report (missing / thin / competitive / strong) | ~$0.01–0.05 per run |
| [`deep-sea.ipynb`](deep-sea.ipynb) | BERTopic + full LLM synthesis | No cluster cap, coverage quality assessment (differentiated / authoritative / redundant), cannibalization flags, Screaming Frog pre-computed embeddings as an input option | A few dollars, scales with site size |

**Rising Tide** uses K-Means, which needs you to guess a cluster count up front. **Open Water** and **Deep Sea** switch to BERTopic, which discovers the number of clusters from the data itself and gives each one a readable auto-generated name. That's the real jump between tiers; the LLM step on top is what turns clusters into a report you can act on.

## What you need

- A CSV export of your site (Screaming Frog or a custom crawler)
- A CSV export of a competitor's site, same format
- For Open Water or Deep Sea: an OpenAI or Anthropic API key

Screaming Frog's default export works with zero configuration: crawl, **File → Export**, make sure you're on the **Internal** tab with the **HTML** filter active, save as CSV. Using a different crawler or a custom dataset instead is fine, you just remap the column names in the configuration cell. Full details, including exactly which columns matter and what a good export looks like, are in [WALKTHROUGH.md](WALKTHROUGH.md).

## Running one

1. Open the notebook in Colab: **File → Open notebook → GitHub**, then paste this repo's URL.
2. **File → Save a copy in Drive.** You want your own copy so your edits stick.
3. Edit the configuration cell: file paths, column names, site labels.
4. **Runtime → Run all.**
5. Come back in a few minutes.

## Tuning

**Rising Tide**

* `N_CLUSTERS` (default 10): how many clusters K-Means creates. Rule of thumb: `sqrt(total_pages / 2)`, adjusted after your first run.

**Open Water / Deep Sea**

* `MIN_TOPIC_SIZE` (default 5): the minimum number of pages needed to form a cluster. Lower it for more, smaller clusters; raise it if you're getting too many two- or three-page clusters. A good starting point is 2–5% of your total page count, floor of 5.
* `N_TOP_WORDS` (default 8): how many keywords BERTopic surfaces per cluster for auto-labeling.
* `MAX_LLM_CLUSTERS` (Open Water only, default 15): caps how many clusters get sent to the LLM, sorted by gap ratio. This is the spend control.

The [tuning guide in WALKTHROUGH.md](WALKTHROUGH.md#tuning-guide) covers what to do when clusters look wrong, when the outlier bucket is too large, and how to filter out boilerplate keywords.

## What you get back

Every tier renders a breadth/depth chart and a UMAP topic map inline, then exports CSVs:

* `all_pages_clustered.csv`: every page from both sites, with its cluster assignment
* Rising Tide: `competitor_gap_summary.csv`, cluster counts and gap ratios
* Open Water: `competitor_gap_report.csv`, prioritized gap report with LLM-written labels and recommended actions
* Deep Sea: `competitor_gap_report.csv`, the full report plus coverage quality flags and similarity scores

## Reading the output

A cluster where the competitor has five times your page count is not automatically five times more urgent. Check whether the topic actually matters to your business before you build a content calendar around it. Deep Sea's coverage quality flags matter here too: a "redundant" flag inside your own cluster usually means consolidating existing pages will move the needle faster than publishing new ones.

## Your data stays yours

Rising Tide runs entirely inside your Colab session: nothing about your site or a competitor's leaves the notebook except the files you explicitly download. Open Water and Deep Sea send only the clustered, already-anonymized-by-cluster content to whichever LLM provider you configure, for the synthesis step alone.

Clear your outputs before sharing a copy of a notebook you have run. Charts and printed cluster tables embed real page data directly in the `.ipynb` file. Use **Edit → Clear all outputs**, then save, before you share a link, email the file, or commit it to a public repo. The same applies to Colab's link sharing: "Anyone with the link" exposes whatever is currently sitting in the output cells.

## License

Released under the [MIT License](LICENSE). Free to use, copy, modify, and redistribute, including commercially. Provided as is, with no warranty.
