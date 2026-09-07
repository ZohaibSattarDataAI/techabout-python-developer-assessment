# TechAbout Python Developer Assessment

Python implementation for the TechAbout assessment, focused on the TECHi.com article metadata reader and offline parser tests.

## Overview

This project implements a small, respectful metadata scraper for published articles on [TECHi.com](https://www.techi.com/).

The scraper:

* Reads `robots.txt` before crawling.
* Discovers sitemap URLs from `robots.txt`.
* Does not hardcode article URLs.
* Handles sitemap indexes and nested XML sitemaps.
* Discovers up to 20 article URLs.
* Checks robots.txt permissions before making requests.
* Uses a real browser User-Agent.
* Limits network requests to approximately one request per second.
* Stores downloaded responses in a local disk cache.
* Handles HTTP errors, timeouts and request failures without terminating the whole run.
* Extracts article metadata using multiple fallbacks.
* Converts absolute and relative dates into ISO-8601 format.
* Writes the final metadata to `techi_articles.csv`.

The project also contains offline tests for date, money and domain parsing.

---

## Project Structure

```text
techabout-python-developer-assessment/
│
├── techi_audit.py
├── techi_articles.csv
├── tests/
│   ├── test_dates.py
│   ├── test_money.py
│   └── test_domains.py
│
├── techi_cache/
│   └── *.cache
│
├── README.md
└── NOTES.md
```

`techi_cache/` is generated automatically when the scraper runs and should normally be excluded from Git with `.gitignore`.

---

## Requirements

* Python 3.10+
* Internet connection for the scraper
* Free/open-source Python packages

Install dependencies with:

```bash
pip install requests beautifulsoup4 pandas pytest
```

---

## Running the Scraper

Run:

```bash
python techi_audit.py
```

The scraper will:

1. Load `https://www.techi.com/robots.txt`.
2. Read sitemap declarations from the robots file.
3. Process the sitemap index and nested sitemap files.
4. Identify article URLs.
5. Select up to 20 articles.
6. Fetch permitted pages with a one-second request interval.
7. Cache successful responses locally.
8. Extract metadata.
9. Save the results to:

```text
techi_articles.csv
```

The output columns are:

```text
url
slug
title
category
author_handle
date_text
date_iso
```

---

## Robots.txt Decision

Article discovery starts from `robots.txt` rather than from a hardcoded list of article URLs.

This was chosen because the assessment specifically requires URL discovery from:

```text
https://www.techi.com/robots.txt
```

The sitemap URLs declared there are then processed recursively where necessary.

Before requesting a URL, the scraper uses Python's `RobotFileParser` to check whether the configured User-Agent is allowed to access it.

If robots.txt cannot be loaded, the scraper fails closed and does not continue crawling.

This avoids silently bypassing the site's crawling rules.

---

## Request Rate

The scraper waits when necessary so that network requests are approximately one second apart.

This is intentionally conservative because the task only requires up to 20 articles. There is no practical reason to make the crawler aggressive.

Cached responses do not require another network request, so rerunning the scraper can be much quieter.

---

## Disk Cache

Downloaded responses are stored in:

```text
techi_cache/
```

The filename is generated from a SHA-256 hash of the URL.

For example:

```text
https://www.techi.com/example-article/
```

is converted into a deterministic cache filename.

The cache was added for two reasons:

1. Avoid downloading the same page repeatedly during development.
2. Make subsequent runs close to silent when the same URLs are requested again.

The cache contains response bodies only; no credentials or private information are stored.

---

## Article URL Discovery

The scraper does not maintain a manually written list of TECHi article paths.

Instead:

```text
robots.txt
     ↓
sitemap URL
     ↓
sitemap index / XML sitemap
     ↓
article URLs
     ↓
article filtering
```

The URL filter rejects obvious non-article paths such as:

```text
/category/
 /tag/
 /author/
 /about/
 /contact/
 /search/
```

It also rejects URLs containing file extensions and URLs outside the TECHi domain.

This keeps the crawler focused without requiring a hardcoded article catalogue.

---

## Metadata Extraction

The scraper uses several fallbacks because web page markup is not guaranteed to be identical across articles.

### Title

The extraction order is approximately:

1. JSON-LD `headline`
2. JSON-LD `name`
3. `<h1>`
4. HTML `<title>`

### Category

The scraper checks:

1. JSON-LD `articleSection`
2. `article:section` metadata
3. Category/topic/section links

### Author

The scraper first looks for an author URL such as:

```text
/author/example/
```

and extracts:

```text
example
```

It then falls back to JSON-LD author information and finally the HTML author meta tag.

The reason for preferring an author URL is that it can provide a stable handle rather than only a display name.

### Date

The scraper checks:

1. JSON-LD
2. `<time>` elements
3. common metadata fields
4. elements with date-related CSS classes

The first candidate that can be successfully parsed is used.

---

## Date Handling

The date parser supports both absolute and relative values.

Examples include:

```text
2026-08-22
2026-08-22T10:30:00Z
August 22, 2026
22 August 2026
Updated 6 days ago
Published 2 hours ago
10 minutes ago
yesterday
today
```

Relative dates require a reference time.

For example:

```text
Updated 6 days ago
```

is converted relative to the time at which the scraper processes the page.

The resulting `date_iso` value is written in UTC using:

```text
YYYY-MM-DDTHH:MM:SSZ
```

For months and years expressed relatively, the implementation uses fixed approximations of 30 days per month and 365 days per year. This is a deliberate simple interpretation rather than pretending that relative website text provides an exact calendar duration.

---

## Error Handling

A single problematic page should not stop the complete run.

The scraper handles:

* timeouts
* connection/request errors
* HTTP 404 responses
* other HTTP failures
* malformed XML
* malformed JSON-LD
* missing metadata
* unexpected HTML structures
* extraction exceptions

When an individual article cannot be fetched or parsed, the scraper continues with the remaining discovered URLs.

Missing metadata is represented by an empty value rather than causing the entire job to fail.

---

## Testing

The parser logic is intended to be tested offline so that tests do not depend on TECHI.com being available.

Run:

```bash
pytest -q
```

The tests cover awkward inputs for:

### Date parsing

Examples:

```text
25/12/2025
12/25/2025
2026-08-22
Updated 6 days ago
impossible dates
blank values
```

### Money parsing

Examples:

```text
Rs. 4,500
PKR 12,500
(500)
$45.00
25k PKR
blank values
```

### Domain parsing

Examples:

```text
https://www.Example.COM/cpanel
www.example.com
example.com:8080
example.com?x=1
invalid values
blank values
```

These tests are intentionally small and focused on parser behaviour rather than testing the live website.

---

## Part A Dataset Note

The assessment instructions referenced a supplied `renewals_raw.csv` containing 34 renewal records.

That dataset was not included with the assessment email.

TechAbout recruitment subsequently confirmed in writing that the file was accidentally omitted and that this was an internal issue.

Therefore, I did **not** create a substitute dataset or fabricate Part A results. Doing so would make the resulting cleaning totals impossible to verify against the actual assessment records.

The Part A work should be run against the real `renewals_raw.csv` once supplied.

---

## Decisions and Trade-offs

### Why Requests + BeautifulSoup?

The task is a relatively small metadata extraction job. `requests` provides straightforward HTTP handling and `BeautifulSoup` is sufficient for parsing the HTML.

A browser automation framework would add complexity without being necessary for the specified task.

### Why XML parsing from the standard library?

Sitemaps are XML documents, so Python's built-in `xml.etree.ElementTree` is sufficient. An additional XML dependency was not necessary.

### Why multiple extraction fallbacks?

Web markup can change or differ between pages. Using JSON-LD first and then falling back to common HTML metadata makes the extractor more tolerant of minor markup differences.

### Why fail closed when robots.txt is unavailable?

Continuing to crawl without knowing whether the site permits access would contradict the requirement to respect robots.txt.

### Why use a cache instead of a database?

Only up to 20 articles are required. A simple file-based cache is enough and avoids unnecessary infrastructure.

### Why not use asynchronous requests?

The assessment explicitly requires approximately one request per second and only asks for up to 20 articles. Async crawling would add complexity without providing a meaningful benefit for this task.

---

## What I Would Improve

If this were being developed beyond the assessment, I would consider:

* Adding cache metadata such as timestamps and HTTP status.
* Adding configurable cache expiry.
* Making the request interval configurable from the command line.
* Adding structured logging instead of console prints.
* Adding retry/backoff handling for temporary server errors.
* Separating the scraper, parsers and output code into small modules.
* Adding fixture-based tests for representative HTML/JSON-LD article pages.
* Adding a command-line option for the article limit.
* Adding a `--no-cache` option for explicit fresh runs.
* Adding validation for required output fields before writing the CSV.

These improvements were intentionally not all implemented because the assessment values a small, understandable solution over unnecessary infrastructure.

---

## Security and Privacy

No TechAbout credentials, private client data, API keys or secrets are required by this project.

The scraper only accesses publicly available TECHI.com pages required for the assessment.

Generated cache files and local development artifacts should not be committed if they contain unnecessary downloaded content.

---

## Summary

The implementation prioritizes:

* respecting `robots.txt`
* minimal network traffic
* reproducible URL discovery
* graceful failure
* tolerant metadata extraction
* offline-testable parsing
* simple implementation
* explainable engineering decisions

The goal is not to build a general-purpose crawler. It is to provide a small, reliable solution to the specific metadata extraction task in the assessment.
