---
layout: post
title: "How to extract clean article text from any URL"
summary: "Strip the nav, ads, and cookie banners from a web page and get just the article body — in one HTTP call, no scraper to maintain."
description: "Extract the readable article text from any URL with a single API call. Working curl, Python, and JavaScript examples, plus how to handle paywalls, redirects, and non-article pages."
reading_time: 5
---

Every text pipeline starts the same way: you have a URL, and you need the words
on it. Not the navigation, not the cookie banner, not the "related stories"
rail — the article.

Doing this yourself means writing a scraper, and scrapers rot. Each site nests
its content differently, and the markup changes without warning. The
`/text` endpoint does the extraction for you.

## The one-call version

```bash
curl "https://use.textapi.com/text?url=https://en.wikipedia.org/wiki/Natural_language_processing"
```

You get back a single JSON field:

```json
{
  "text": "Natural language processing - Wikipedia : \n\nNatural language processing has its roots in the 1950s. Already in 1950, Alan Turing published an article titled \"Computing Machinery..."
}
```

That call returns about 42,000 characters of article body from a page whose
raw HTML is many times larger.

## The response format, precisely

The `text` field is the page **title**, then `" : "`, then the **article
body**. That separator is worth knowing, because it means you can split the
title back off when you need it:

```python
import requests

r = requests.get(
    "https://use.textapi.com/text",
    params={"url": "https://en.wikipedia.org/wiki/Natural_language_processing"},
)
title, _, body = r.json()["text"].partition(" : ")

print(title)          # Natural language processing - Wikipedia
print(len(body))      # 42554
```

In JavaScript:

```js
const res = await fetch(
  "https://use.textapi.com/text?url=" +
    encodeURIComponent("https://en.wikipedia.org/wiki/Natural_language_processing")
);
const { text } = await res.json();
const [title, ...rest] = text.split(" : ");
const body = rest.join(" : ");
```

Note the `join` in that last line — a title containing `" : "` would otherwise
lose part of the body. Python's `partition` splits on the first occurrence only,
so it doesn't have that problem.

## POST it instead when the URL is long

Query strings have practical length limits, and URLs with many parameters need
careful escaping. The `POST` form takes JSON and behaves identically:

```bash
curl -X POST "https://use.textapi.com/text" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://en.wikipedia.org/wiki/Natural_language_processing"}'
```

## What to expect when it doesn't work

Extraction is a heuristic, not magic. A few cases to plan for:

**Pages that aren't articles.** A homepage, a search results page, or a product
grid has no single body of prose to find. You'll get *something* back — usually
the longest block of text — but it won't be meaningful. Check the length before
trusting it.

**Paywalls.** If a site serves a teaser paragraph to anonymous visitors, that's
what gets extracted, because that's what the page contains.

**Pages that need JavaScript.** Content injected by client-side rendering isn't
in the HTML the API receives, so it can't be extracted.

**Unreachable URLs.** A host that doesn't resolve, times out, or returns an
error gives you a `502` with a message, rather than a partial result. Private
and internal addresses are rejected with a `400` — the endpoint only fetches
public hosts.

A reasonable guard in production:

```python
resp = requests.get("https://use.textapi.com/text", params={"url": url})

if resp.status_code != 200:
    raise RuntimeError(f"extraction failed: {resp.json().get('detail')}")

text = resp.json()["text"]
if len(text) < 500:
    # Probably not an article — skip it rather than feeding noise downstream.
    ...
```

## Chaining it into the rest of the pipeline

Extraction is rarely the goal on its own. Once you have clean text, the other
endpoints take it directly:

```python
text = requests.get(
    "https://use.textapi.com/text", params={"url": url}
).json()["text"]

entities = requests.get(
    "https://use.textapi.com/ner", params={"text": text[:5000]}
).json()["spans"]
```

Two things to note there. First, entity recognition on a 42,000-character
article is slower and rarely more useful than running it on the opening
section, so trimming is usually the right call. Second, if what you actually
want is an answer rather than a pile of entities, `/qa` accepts a `url`
directly and does the extraction step for you — no need to make two calls.
