# Web Crawler — Research Data Collection Infrastructure

[Algorithm guide: pseudocode, time complexity, and memory](docs/ALGORITHM_GUIDE.md).

[![CI](https://github.com/fatehaszaman/web-crawler/actions/workflows/ci.yml/badge.svg)](https://github.com/fatehaszaman/web-crawler/actions/workflows/ci.yml)

### Java/Python crawl tooling for IEEE ICDCS 2020 website-fingerprinting research

This repository contains the URL-collection tooling I built for a university Network Privacy Lab. It fed the deep-learning website-fingerprinting pipeline whose results were published at IEEE ICDCS 2020. The job of this code is narrow but load-bearing: produce large, clean, domain-scoped URL sets that downstream packet-capture and CNN/LSTM training stages can consume without manual cleanup.

## Information Systems & Scalability

* **Bulk URL acquisition.** `WebCrawler.java` is a BFS crawler that ingests up to a configurable cap (run at 50,000 URLs per invocation) using a `Queue` of frontier URLs plus a `HashSet` of visited URLs to prevent revisits and unbounded growth.
* **Domain scoping and content filtering.** The crawler enforces a strict `https://www.nytimes.com/` prefix check and a configurable exclusion list (section keywords, topic paths, Wirecutter links) so that downstream training data stays on-distribution and free of low-value sections (interactives, crosswords, opinion tags, etc.).
* **Deduplication discipline at every stage.** `URLScraper.java`, `UniqueLinks.java`, and `PrintLinksInAlphabeticalOrder.java` all apply `HashSet` deduplication and pattern-based validation (`^https://www\.nytimes\.com/\d{4}/\d{2}/\d{2}/[a-z\-]+\.html$`) before any URL is allowed into the dataset. This is what keeps the eventual trace dataset clean without a separate cleanup pass.
* **Sampling vs. exhaustive modes.** Two collection strategies are supported: BFS traversal (`WebCrawler.java`) for broad coverage from a seed URL, and homepage random-sampling (`URLScraper.java`, capped at 1,000 unique article URLs per run) for distributional balance. These two strategies are where the lab's collection approach started; the larger lab corpus came from the later version described in "Where this repository fits."

## SDLC & Engineering Discipline

* **Versioned iteration of the collection logic.** `Web-Crawler-Trial-1.py` through `Web-Crawler-Trial-4.py` preserve the Python prototype generations (Wikipedia random pages, USA Today, BBC, BBC-with-dedup) that fed into the final Java implementation. Keeping these in-repo makes the design decisions auditable rather than folkloric.
* **Small, single-purpose tools.** Each file does one job (crawl, sample, generate, dedupe, sort). Composition happens via files on disk, which kept the lab pipeline debuggable when a single stage misbehaved.
* **Reproducibility primitives.** Seed URL, max-URL cap, and Wirecutter-exclusion flag are explicit parameters on `WebCrawler.crawl(...)`, so every dataset run can be reproduced from three values.

## Impact

The output of this tooling was the input to the lab's packet-capture and traffic-trace pipeline, which trained CNN/LSTM models published at **IEEE ICDCS 2020**. Reworking the crawl logic produced roughly a 60% improvement in output volume, validity, and run time over the prior approach the lab had been using.

## How to Run

Requirements: JDK 8+ for the Java crawlers; Python 3.8+ with `requests`, `beautifulsoup4`, and `lxml` for the prototypes.

**Java — main crawler:**

```bash
javac WebCrawler.java
java WebCrawler
```

To change the seed URL, max URL cap, or Wirecutter exclusion, edit the constants in `main(...)` at the bottom of `WebCrawler.java`. Output is printed to stdout one URL per line, so the usual pattern is to redirect: `java WebCrawler > links.txt`.

**Java — homepage sampler:**

```bash
javac URLScraper.java && java URLScraper > sampled.txt
```

**Java — post-processing:**

```bash
javac UniqueLinks.java && java UniqueLinks                 # reads ./Links.txt
javac PrintLinksInAlphabeticalOrder.java                   # edit filePath in source first
java PrintLinksInAlphabeticalOrder
```

**Python prototypes:**

```bash
pip install requests beautifulsoup4 lxml
python Web-Crawler-Trial-1.py
```

## Files

| File | Role |
| --- | --- |
| `WebCrawler.java` | BFS crawler with visited-set, domain filter, and exclusion-list. Primary tool. |
| `URLScraper.java` | Homepage random-sampler; collects up to 1,000 unique article URLs matching the NYT date/section pattern. |
| `NYTimesURLGenerator.java` | Probes URL patterns by HTTP response code to surface valid endpoints. |
| `PrintLinksInAlphabeticalOrder.java` | Post-processor: extracts `href` targets from a file, sorts, reports count. |
| `UniqueLinks.java` | Deduplication pass over raw link output; filters to valid NYT `.html` URLs. |
| `Web-Crawler-Trial-1..4.py` | Python prototypes preserved for design provenance. |

## Stack

Java (`java.net.URL`, `java.util.regex`, BFS over `Queue`/`HashSet`), Python (`requests`, `BeautifulSoup`, `lxml`), regex-based HTML parsing.

## Where this repository fits

This is the early version of the URL-collection tooling I started with for Dr. Chan-Tin's network-privacy lab, kept here to show how the design began: a BFS crawler, a homepage sampler, and URL validation and deduplication, run single-threaded with a 50,000-URL cap per run. The full lab version grew from this starting point into a multithreaded Java crawler that processed 300,000+ URLs per site, with sorting and cleaning in Java, Python and terminal tools, across 1,000+ Docker runs. That version was not published here.

## Roadmap / Future Hardening

The public code in this repo is single-process and single-threaded, with `e.printStackTrace()` as the error path. Next steps for hardening this public version into a production-grade ingestion service:

* Concurrent fetching via a thread pool or an async client (Java `HttpClient` with `CompletableFuture`, or Python `asyncio` + `aiohttp`) to saturate I/O.
* A retry-with-exponential-backoff wrapper around fetch, distinguishing rate-limit (429), transient (5xx, timeouts), and permanent (404) failures.
* Structured persistence (Postgres or SQLite) replacing stdout/text-file handoff between stages, with a `(url, fetched_at, status, content_hash)` schema for resumability.
* A containerized run target (Dockerfile) plus a CI workflow that runs unit tests on the URL filter and dedup logic to prevent regressions on the parser.

These are not claimed as present in the public code in this repository — they are the IS-SDLC roadmap for hardening this research tooling into infrastructure.
