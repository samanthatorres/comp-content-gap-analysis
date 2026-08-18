# Competitor Gap Analysis via Topic Clustering

### A Walkthrough Guide

*Sam Torres — The SEO Mermaid*

## Before you open the notebook

This guide explains what the notebook is doing and why — so you understand the decisions being made at each step, not just how to run the code.

If you just want to run it and see results, skip to [Quick Start](#quick-start). But if you want to understand the framework well enough to adapt it to your own problems, read through this first.

## Preparing your data

This section tells you exactly what your CSV files need to look like before you open the notebook. Get this right first and everything else runs smoothly.

### This notebook needs two CSV files

One for your site. One for your competitor. Same format for both.

### Option A: Screaming Frog export

This is the recommended path. Screaming Frog's default crawl export gives you everything the notebook needs with no additional setup.

**How to export:**

1.  Run a crawl of your site (and separately, your competitor's site)

2.  In the top menu: File > Export — this exports the current view as CSV

3.  Make sure you're in the **Internal** tab with **HTML** filter active before exporting — this ensures you're only exporting HTML pages, not images, CSS, JS, or PDFs

4.  Save as .csv

**What the export contains (Screaming Frog defaults):**

| Column | What it is | Required? |
|---|---|---|
| `Address` | Full URL of the page | ✅ Required |
| `Title 1` | Page title tag | ✅ Required |
| `Meta Description 1` | Meta description | ✅ Strongly recommended |
| `H1-1` | First H1 heading | ✅ Strongly recommended |
| `Status Code` | HTTP status (200, 301, 404, etc.) | 👍 Nice to have — filters to live pages only |
| `Content Type` | MIME type (text/html, image/jpeg, etc.) | 👍 Nice to have — filters to HTML pages only |
| `Word Count` | Approximate word count | Optional |
| `Indexability` | Whether the page is indexable | Optional |

**The notebook uses Status Code and Content Type to automatically filter out redirects, error pages, images, and other non-HTML resources. If those columns aren't present, the filter is skipped and all rows are included — which means you may want to pre-filter your export manually.**

**Bare minimum:** Address + Title 1. The notebook will run with just these two columns. Output quality will be lower than with meta descriptions and H1s included.

**Recommended setup:** Address + Title 1 + Meta Description 1 + H1-1 + Status Code + Content Type. This is the default Screaming Frog export — you don't need to add anything.

**What a good export looks like:**

```csv
Address,Content Type,Status Code,Title 1,Meta Description 1,H1-1
https://example.com/blog/seo-guide,text/html,200,The Complete SEO Guide,Everything you need to know about SEO in 2024.,The Complete SEO Guide
https://example.com/blog/keyword-research,text/html,200,Keyword Research for Beginners,How to find and prioritize keywords for your content.,Keyword Research for Beginners
https://example.com/images/logo.png,image/png,200,,,
```

The third row (an image) will be automatically filtered out. Only the first two rows make it into the analysis.

### Option B: Custom CSV

If you're using a different crawler, pulling data from an API, or have your own dataset, you can use any CSV — you just need to map your column names in the config cell.

**Bare minimum (notebook will run but output will be basic):**

| Column | What it contains | Example value |
|---|---|---|
| A URL column | Full page URL | `https://example.com/page` |
| At least one text column | Any descriptive text about the page | Page title, H1, description |

**Recommended (better embeddings, better clusters):**

| Column | What it contains | Example value |
|---|---|---|
| URL | Full page URL | `https://example.com/page` |
| Title | Page title | The Complete SEO Guide |
| Description | Meta description or summary | Everything you need to know… |
| Heading | Main heading or H1 | The Complete SEO Guide |

**What a good custom CSV looks like:**

```csv
url,title,description,heading
https://example.com/blog/seo-guide,The Complete SEO Guide,Everything you need to know about SEO.,The Complete SEO Guide
https://example.com/blog/keyword-research,Keyword Research for Beginners,How to find keywords for your content.,Keyword Research
```

**Then in the config cell, update these lines:**

URL_COLUMN = "url" # whatever your URL column is named

TEXT_COLUMNS = ["title", "description", "heading"] # whichever text columns you have

### What makes a good text input for embeddings

The embedding model reads the text you give it and converts it into a semantic vector. Better text = better vectors = better clusters. Here's what to aim for:

**Good text signals:**

-   Page titles — usually the clearest statement of what a page is about

-   Meta descriptions — often add secondary context and related concepts

-   H1 headings — usually match or complement the title

-   Body copy excerpts — if you have them, the first 200-300 words work well

**Noise to avoid if possible:**

-   Navigation labels ("Home", "About", "Contact")

-   Footer boilerplate ("© 2024 Company Name. All rights reserved.")

-   Cookie consent text

-   Template phrases repeated across all pages ("Read more", "Learn more", "Get started")

The Screaming Frog defaults (Title + Meta Description + H1) strike a good balance — they're descriptive without including too much boilerplate. If you find your clusters look messy or generic, check whether your titles or meta descriptions are templated across many pages.

### Checking your CSV before you run

A few things worth verifying before you start:

1.  **Open your CSV in a spreadsheet app** and confirm the column names match exactly what you've set in URL_COLUMN and TEXT_COLUMNS. Column names are case-sensitive — Title 1 and title 1 are different.

2.  **Check that URLs are full absolute URLs**, not relative paths. The notebook expects https://example.com/page, not /page.

3.  **Check for empty rows or header rows repeated mid-file.** Large Screaming Frog exports sometimes have summary rows or duplicate headers. These will cause errors. Delete them before uploading.

4.  **Check your page count is roughly right.** If your site has 300 pages and your export has 2,000 rows, you probably exported everything (including images, CSS, JS) rather than HTML-only. Re-export with the HTML filter active, or add Status Code and Content Type columns so the notebook can filter automatically.

5.  **For competitors: you don't need a complete crawl.** A sample of their most important pages is enough for topic modeling. 100--500 pages is plenty for most sites. You can pull a top-pages-by-traffic report from Ahrefs or Semrush and use that instead of a full crawl.

## The problem we're solving

Traditional competitor gap analysis gives you a keyword list. You export your rankings, export theirs, find the delta, and end up with a spreadsheet of keywords you're not targeting. That's useful. But it answers the wrong question.

A keyword list tells you *what* you're missing. It doesn't tell you:

-   How deeply a competitor owns a topic — whether they have two pages on something or twenty

-   Whether the gaps in your content are structural (entire topic areas you're not covering) or tactical (individual keywords within topics you do cover)

-   Where you're actually strong — which topic territories you could defend or double down on

This notebook answers a better question:

> **"Who owns this topic space, and where are the gaps in breadth and depth?"**

It does that by treating content as *structure* rather than a list of keywords, using three ML layers to map the topic landscape across both sites simultaneously.

## The three-layer framework

### Layer 1: Embeddings — *What is similar?*

An embedding converts text into a vector — a list of numbers that captures its semantic meaning. Two pieces of content about the same thing will have vectors that are close together in vector space, even if they use completely different words.

This is the critical difference from keyword matching. A page about "content freshness" and a page about "keeping articles up to date" would match on almost no keywords, but their embeddings would be very close — because they're about the same idea.

Once your content is embedded, similarity is *measurable*, not a judgment call. You can calculate exactly how close any two pages are, across your site and your competitor's, without reading a single one of them manually.

### Layer 2: Clustering with BERTopic — *What belongs together?*

Once pages are in vector space, clustering groups the ones that belong together. This notebook uses **BERTopic** — and there's a specific reason for that choice.

Most clustering tutorials reach for k-means. K-means is simple and fast, but it has a meaningful limitation: you have to decide the number of clusters before you run it. If you have 200 pages and you ask for 10 clusters, you get 10 clusters — even if the data really wants to form 7 or 14. You're imposing structure rather than discovering it.

BERTopic discovers structure from the data itself. Here's how:

**UMAP** first reduces the high-dimensional embeddings (384 dimensions from Sentence Transformers) down to 5 dimensions. This makes the math faster and preserves the neighborhood structure of the data — pages that are close in 384D are still close in 5D.

**HDBSCAN** then finds dense regions in that reduced space. Unlike k-means, which assigns every point to a cluster, HDBSCAN identifies genuinely dense clusters and assigns everything else to an outlier bucket (cluster -1). This is important: it means you're not forcing thin or ambiguous pages into topic groups where they don't belong.

**c-TF-IDF** then automatically labels each cluster with representative keywords by measuring which terms are most distinctive to that cluster versus the rest of the corpus. This gives you a head start on understanding what each cluster is about before you even read the pages.

**The result:** You get a set of topic clusters that reflects the actual structure of the content — not a structure you imposed on it. And each cluster comes pre-labeled with keywords, which makes both the sanity-check step and the LLM synthesis step more effective.

### Layer 3: LLM Synthesis — *What does it mean?*

BERTopic gives you clusters and keywords. The LLM interprets them into strategy. For each cluster, we send the BERTopic keywords plus a sample of pages to the LLM and ask it to:

-   Name the topic cluster (refining or confirming BERTopic's auto-label)

-   Assess the coverage gap (missing, thin, competitive, or strong)

-   Assign a priority level

-   Recommend one concrete action

Because BERTopic already provides keywords, the LLM prompt in this notebook is richer than it would be with k-means alone — the model has explicit topic signal to work with, not just raw page content. The labels it produces are correspondingly more precise.

## Step-by-step walkthrough

### Step 0: Install dependencies

sentence-transformers — the embedding model (free, local)

bertopic — topic modeling and keyword extraction

hdbscan — density-based clustering (used inside BERTopic)

scikit-learn — CountVectorizer for BERTopic's keyword extraction

umap-learn — dimensionality reduction (used inside BERTopic and for visualization)

pandas / numpy — data handling

matplotlib / seaborn — charts

openai / anthropic — LLM synthesis (only used in Step 8)

The key addition versus a k-means notebook is bertopic and hdbscan. Both install cleanly on Colab with no additional configuration.

### Step 1: Configuration

The only cell you need to edit. Key settings:

**MIN_TOPIC_SIZE** — This is BERTopic's equivalent of k-means' N_CLUSTERS, but it works differently. Rather than setting the number of clusters, you set the *minimum number of pages required to form a cluster*. HDBSCAN then discovers how many clusters that produces naturally.

A good starting point is 2-5% of your total page count, with a floor of 5. If you have 200 pages total, try MIN_TOPIC_SIZE = 5. If you end up with too many tiny clusters, increase it to 8 or 10. If you get only a few very broad clusters, decrease it to 3.

**N_TOP_WORDS** — How many keywords BERTopic surfaces per topic. These appear in the cluster summary and in the LLM prompt. 8 is a good default — enough context to understand the topic, not so many that the prompt gets cluttered.

**TEXT_COLUMNS** — What content gets embedded. For Screaming Frog exports, ["Title 1", "Meta Description 1", "H1-1"] captures the primary topic signals. If you have richer content available (body copy, structured data fields), adding those columns will produce more meaningful embeddings and better clusters.

### Step 2: Load and prepare data

Both CSVs are loaded, filtered to live HTML pages, and combined into one dataset tagged by source. The filtering logic (status code 200, content type text/html) is specific to Screaming Frog exports — if your CSV doesn't have those columns, the filter is skipped automatically.

**What to check:** After loading, confirm the page counts look right for both sites. If one is unexpectedly low, run df.columns.tolist() to see your actual column names and adjust TEXT_COLUMNS or URL_COLUMN in the config.

### Step 3: Generate embeddings

Every page's content is converted to a semantic vector using Sentence Transformers' all-MiniLM-L6-v2. This model produces 384-dimensional vectors and runs entirely locally — no API key, no cost, no data leaving your environment.

The first run downloads the model (~90MB). After that it's cached in your Colab session and subsequent runs are instant.

**Why pass embeddings directly to BERTopic?** BERTopic can run its own embedding step internally, but by pre-computing embeddings and passing them in, we get two advantages: we can swap in OpenAI embeddings without changing anything downstream, and if we need to re-run BERTopic with different settings (adjusting MIN_TOPIC_SIZE, for example), we don't have to re-embed all the pages from scratch.

**The OpenAI alternative** is commented out in the code. If you want higher-quality embeddings and have an API key, uncomment that block, set EMBEDDING_PROVIDER = 'openai', and add your key. The rest of the notebook is unchanged.

### Step 4: BERTopic clustering

This is the most technically involved step, but most of the complexity is handled by the library. Three sub-models run in sequence:

**UMAP configuration:**

-   n_components=5 — reduces to 5 dimensions before clustering (BERTopic's standard approach; 2D would lose too much structure for clustering, even though we use 2D for visualization later)

-   min_dist=0.0 — keeps similar points tightly packed, which helps HDBSCAN find dense clusters

-   metric='cosine' — matches how the embeddings were trained

**HDBSCAN configuration:**

-   min_cluster_size=MIN_TOPIC_SIZE — the minimum density threshold for a cluster

-   min_samples=1 — controls how conservative the outlier assignment is; setting this to 1 means HDBSCAN is relatively generous about including pages in clusters rather than calling them outliers. Increase it if you're getting unexpected cluster assignments; decrease it if too many pages are landing in -1.

-   prediction_data=True — required for soft cluster assignment (BERTopic uses this internally)

**CountVectorizer configuration:**

-   ngram_range=(1, 2) — captures both single-word and two-word keyword phrases, which are often more descriptive (e.g. "content strategy" rather than just "content")

-   min_df=2 — a keyword must appear in at least 2 pages to be included, which filters out one-off terms that don't characterize the cluster

**About the outlier cluster (-1):** Pages that HDBSCAN can't confidently assign to any cluster go into -1. These are not failed pages — they're pages that sit at topic boundaries or are genuinely unique on your site. They're excluded from the gap analysis but included in the final export so you can review them. A high outlier rate (over 20%) usually means MIN_TOPIC_SIZE is set too high, or your content is more diverse than the clustering can handle at that granularity.

**Reading the topic summary table:** The Name column shows BERTopic's auto-generated label, which is a concatenation of the top keywords. Something like 0_content_strategy_seo_optimization tells you cluster 0 is about content strategy and SEO. The Count column is total pages across both sites in that cluster.

### Step 5: Gap summary

The gap summary table breaks each topic cluster down by site and calculates gap_ratio — the ratio of competitor pages to your pages. A ratio of 5.0 means the competitor has five times your coverage in that topic. A ratio of 0.2 means you have five times their coverage.

The table is sorted by gap_ratio descending, so the biggest competitor advantages appear at the top. These are your highest-priority gaps.

**The two types of gaps:**

-   **Breadth gaps:** Topics where you have 0 pages. These appear as gap_ratio values calculated against a floor of 0.1 (to avoid division by zero), so they'll show as very large numbers. In the LLM synthesis step, these get classified as "missing."

-   **Depth gaps:** Topics where you have some pages but the competitor has significantly more. These show as elevated but not extreme gap_ratio values, and get classified as "thin."

### Step 6: Breadth & Depth visualization

Two charts:

**The depth chart:** Side-by-side bar chart showing page counts per cluster. The x-axis shows truncated BERTopic topic names. A cluster where one bar towers over the other is a depth gap.

**The breadth heatmap:** Normalizes page counts within each site (so a small site and a large site are comparable) and shows relative coverage as a color gradient. Dark red = strong coverage. Light yellow = weak coverage. A row where your site is yellow and the competitor is red is a breadth gap, regardless of absolute page counts.

**Saving the charts:** Both are saved as PNG files in your Colab session. Download them from the file browser on the left sidebar, or update the save paths to write directly to Google Drive.

### Step 7: UMAP topic space map

BERTopic ran UMAP internally at 5 dimensions for clustering. Here we run a separate UMAP at 2 dimensions purely for visualization — so we can see the topic landscape as a 2D map.

Each dot is a page. Circles (○) are your pages; triangles (△) are the competitor's. Colors match. Outlier pages are shown in grey behind everything else.

Cluster labels are placed at the centroid of each cluster, using BERTopic's auto-generated topic names (truncated to 22 characters for readability).

**What to look for:**

-   Areas of the map where you see almost exclusively competitor triangles — topic territories they've built out that you haven't touched

-   Areas where circles and triangles are well mixed — competitive topic areas

-   Areas where you have dense circle coverage and few triangles — your strongest positions

-   Isolated grey dots far from any labeled cluster — pages that may be orphaned or sitting at topic boundaries worth investigating

### Step 8: Sample pages per cluster

Before the LLM synthesis step, this cell prints a preview of each cluster: BERTopic's keywords, the page count breakdown, and a sample of actual page content strings.

Read through this output before running Step 8. BERTopic's automatic topic discovery is good, but it's not infallible. Occasionally two genuinely different topics get merged into one cluster, or a cluster's keywords look right but the pages tell a different story. Catching this before the LLM synthesis saves you from labeling errors that carry through to the gap report.

If a cluster looks wrong, your options are:

-   Adjust MIN_TOPIC_SIZE and re-run from Step 4 (fast — no re-embedding needed)

-   Accept the grouping and trust the LLM to label it more accurately than the keywords suggest

-   Manually reassign a small number of obvious outliers in the DataFrame before Step 8

### Step 9: LLM Synthesis — Gap analysis + Coverage quality assessment

This step does two distinct jobs per cluster, in a single LLM call.

**Part 1: Competitive gap analysis** — same as before. The LLM assesses whether each topic cluster represents a missing, thin, competitive, or strong position relative to the competitor.

**Part 2: Coverage quality assessment** — this is new. The LLM looks inward at *your own* pages within the cluster and classifies the coverage as one of three states:

-   **Differentiated** ✅ — pages in this cluster cover clearly distinct angles, subtopics, or intents. This is healthy content architecture. Each page is earning its place.

-   **Authoritative** ⭐ — pages show real depth and breadth with minimal overlap. A genuine strength signal. This is what good topic ownership looks like.

-   **Redundant** ⚠️ — pages are highly similar to each other, likely targeting the same intent with near-duplicate content. This is a cannibalization risk. Having 10 pages in a cluster doesn't mean you own the topic if they're all saying the same thing.

**The ML signal that informs the LLM:** Before the LLM call, the notebook computes cosine similarity across all your pages within each cluster. The mean and maximum similarity scores are passed directly into the prompt. This gives the LLM quantitative evidence, not just the page content to eyeball.

The interpretation guide built into the prompt:

-   Mean similarity below 0.70 — typically differentiated

-   Mean similarity 0.70--0.84 — overlapping, worth reviewing

-   Mean similarity 0.85 and above — likely redundant

The LLM uses this signal alongside the actual content previews to make its classification. It can override the similarity signal if the content clearly tells a different story — for example, if two pages have similar titles but obviously serve different intents.

**Why combine both analyses in one call?** Efficiency, but also coherence. The same cluster context — keywords, page counts, content samples, similarity scores — informs both the gap assessment and the coverage quality assessment. Running them together means the LLM has the full picture for both judgments simultaneously.

**If an LLM call fails:** The notebook catches the error and continues. Failed clusters are skipped. The similarity scores for that cluster are still computed and stored regardless.

### Step 10: Gap Report and Export

The final output now shows two layers per cluster:

**Competitive layer** — priority tier (🔴🟡🟢), gap type (missing/thin/competitive/strong), page counts, and one recommended action.

**Coverage quality layer** — the classification (✅ differentiated / ⭐ authoritative / ⚠️ redundant), the LLM's one-sentence explanation, and a coverage action if one is needed. If your site has no pages in the cluster, this section is omitted.

**The redundancy summary** — at the bottom of the report, any clusters flagged as redundant are surfaced separately as a consolidated cannibalization watchlist. This makes it easy to hand off to a content team: here are the clusters where you should consolidate or differentiate before building more content.

The report is sorted by competitive priority first (high → medium → low), then by gap type (missing → thin → competitive → strong). This means the most urgent competitive gaps appear at the top, and within each priority tier, missing topics come before thin ones.

**Output files:** \| File \| What it contains \| \|—\-\--\|—————\--\| \| competitor_gap_analysis.png \| Breadth & depth charts \| \| topic_space_map.png \| UMAP visualization \| \| competitor_gap_report.csv \| Full report: gap type + coverage quality + similarity scores \| \| competitor_gap_report_unlabeled.csv \| Cluster summary without LLM (if no API key) \| \| all_pages_clustered.csv \| Every page with cluster and topic label \|

The CSV now includes mean_similarity and max_similarity columns alongside the LLM fields — useful for sorting or filtering on the quantitative signal independently of the LLM's classification.

## Tuning guide

Run the notebook once with default settings, then use this guide:

**Too many small clusters (lots of clusters with 2-3 pages)?** Increase MIN_TOPIC_SIZE to 8 or 10 and re-run from Step 4.

**Too few broad clusters (everything lumped together)?** Decrease MIN_TOPIC_SIZE to 3 and re-run from Step 4.

**Too many outlier pages (cluster -1 is huge)?** Lower min_samples in the HDBSCAN config from 1 to… well, 1 is already the minimum. Instead, try decreasing MIN_TOPIC_SIZE so more pages qualify for clusters, or check whether your content is genuinely very diverse.

**Keywords feel generic or unhelpful?** Add domain-specific stop words to the CountVectorizer:

```python
vectorizer_model = CountVectorizer(
    stop_words='english',
    ngram_range=(1, 2),
    min_df=2,
    # Add your own stop words:
    # stop_words=['company', 'blog', 'learn', 'guide', 'best']
)
```

**BERTopic feels slow on a large dataset?** Save your embeddings to disk after Step 3 so you can reload them without re-running the embedding step:

```python
import numpy as np
np.save('embeddings.npy', embeddings)
# To reload: embeddings = np.load('embeddings.npy')
```

## Common issues and fixes

| Issue | Likely cause | Fix |
|---|---|---|
| `KeyError: 'Address'` | Column name mismatch | Check `TEXT_COLUMNS` and `URL_COLUMN` against your CSV headers |
| Most pages in outlier cluster | `MIN_TOPIC_SIZE` too high | Decrease `MIN_TOPIC_SIZE` |
| All pages in one or two clusters | `MIN_TOPIC_SIZE` too low | Increase `MIN_TOPIC_SIZE` |
| BERTopic keywords look generic | Stop words not filtering domain boilerplate | Add custom stop words to `CountVectorizer` |
| LLM synthesis fails | API key missing or expired | Check your key in the config cell |
| Very long runtime | Large dataset | Save embeddings to disk; run on a Colab GPU runtime |

## Quick start

1.  Open the notebook in Google Colab

2.  Upload your CSVs (drag into the left sidebar, or mount Drive)

3.  Edit the CONFIGURATION cell:

    -   Set YOUR_SITE_CSV and COMPETITOR_CSV

    -   Check URL_COLUMN and TEXT_COLUMNS match your CSV headers

    -   Set your site labels

    -   Add an LLM API key if you want labeled output (optional)

4.  Runtime > Run all

5.  Come back in ~5-10 minutes

## What comes next

**Address redundancy before building new content.** If the report surfaces clusters flagged as redundant, fix those first. Consolidating or differentiating near-duplicate pages often has faster ranking impact than publishing new content — and it removes the cannibalization drag that may be suppressing your existing pages.

**Feed it into a content calendar.** High-priority missing clusters are your new content pillars. Each cluster represents a topic territory — not a single keyword — so think in terms of content hubs, not individual posts.

**Run it on a second competitor.** The same notebook works on any pair of sites. Different competitors are often strong in different topic areas, and comparing multiple gap reports will reveal which gaps are truly strategic versus which are just one competitor's idiosyncratic focus.

**Pair it with internal linking.** Once you know which topic territories to build out, a companion notebook using the same embedding-and-clustering framework helps you make sure those new pages get linked into your existing content architecture.

*Sam Torres — The SEO Mermaid*
