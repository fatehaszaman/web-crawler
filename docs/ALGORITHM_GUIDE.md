# Algorithm guide

These cards separate local computation from network latency. They describe
selected Java implementations, not permission to crawl any website or a
guarantee that the crawler is production-ready.

## Filter and deduplicate saved links

Implementation: [`UniqueLinks.main`](../UniqueLinks.java).

```text
# Unique Links / Streaming Input + Hash Set
# Input: saved text file with N lines and B total characters
# Output: qualifying unique URLs printed in unspecified order
# Time: expected O(B), including reading, hashing, and output
# Memory: O(U_chars + L_max), for retained unique URLs and current line

OPEN Links.txt
FOR each line:
    IF it starts with the configured URL prefix AND ends with ".html":
        ADD line to hash set
FOR each unique retained line: PRINT line
CLOSE reader
```

Duplicate input lines are processed but stored once. This is streaming input,
not constant-memory deduplication: U_chars is the total length of retained
unique URLs. The literal prefix test is not a parsed hostname check; a
lookalike hostname sharing that prefix could pass. It is not URL normalization.

## Queue-based crawling

Implementation: [`WebCrawler.crawl`](../WebCrawler.java).

```text
# Crawl / FIFO Discovery with Seen Set
# Input: root URL, discovery limit, exclusion settings
# Output: printed discovered URLs and retained queue/visited state
# Local time: O(sum B_page^2 + sum regex_cost(page) + E*F*L) upper model
# Memory: O(V*L + B_max) logical live data, excluding runtime/network buffers
# V = discovered URLs; E = examined links; F = exclusion terms
# L = URL/term length bound; B_page = page characters

ADD root to queue and visited
WHILE queue not empty AND visited count < limit:
    REMOVE next URL
    READ page line by line, concatenating immutable strings
    FIND links with the configured regex
    FOR each match while discovery limit permits:
        CHECK exclusion substrings, visited set, and accepted URL prefix
        IF accepted: mark visited, PRINT, ENQUEUE
```

Repeated `rawHTML += inputLine` can copy a growing prefix, giving quadratic
page-assembly work in the worst case. Regex matching cost is left explicit:
calling the traversal O(V+E) alone would hide parsing and string work.
Hash operations use an expected-cost model. Network fetches and waiting time
are additional and not bounded by this expression.

The limit counts discovered URLs, not completed successful fetches. Reaching it
can leave queued URLs unread. Visited state is retained between calls; relative
links are not resolved, and query-string variants are not canonicalized.
These notes do not add retries, timeouts, robots handling, or a security policy.
