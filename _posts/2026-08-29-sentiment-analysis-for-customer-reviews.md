---
layout: post
title: "Sentiment analysis for customer reviews, and where it quietly fails"
summary: "Score reviews in one call, read the four numbers correctly, and recognise the sentences this approach scores as neutral when a human wouldn't."
description: "How to score customer review sentiment with one API call: what the compound, pos, neu, and neg values mean, which thresholds to use, and the lexicon limits that make some negative reviews score neutral."
reading_time: 6
---

Sentiment scoring is the rare NLP task with an honest one-line answer:

```bash
curl -G "https://use.textapi.com/sentiment" \
  --data-urlencode "text=Absolutely love this product, works perfectly!"
```

```json
{"neg": 0.0, "neu": 0.308, "pos": 0.692, "compound": 0.8746}
```

The interesting part isn't the call. It's knowing what those four numbers mean
and, more importantly, when to distrust them.

## Reading the four numbers

`pos`, `neu`, and `neg` are the proportions of the text falling into each
category. They sum to 1. `compound` is the single normalised score from −1
(maximally negative) to +1 (maximally positive), and it's the one you should
actually use for decisions.

The conventional thresholds:

```python
def label(compound):
    if compound >= 0.05:
        return "positive"
    if compound <= -0.05:
        return "negative"
    return "neutral"
```

Scoring a batch of reviews:

```python
import requests

reviews = [
    "The delivery was great",
    "This is terrible",
    "Absolutely love this product, works perfectly!",
]

for review in reviews:
    scores = requests.get(
        "https://use.textapi.com/sentiment", params={"text": review}
    ).json()
    print(f"{scores['compound']:+.4f}  {label(scores['compound']):8}  {review}")
```

```
+0.6249  positive  The delivery was great
-0.4767  negative  This is terrible
+0.8746  positive  Absolutely love this product, works perfectly!
```

## Emphasis counts, which is unusual and useful

The scorer is sensitive to the things people actually do when they feel
strongly. Capitalisation and punctuation raise intensity:

```
"amazing"     → compound 0.5859
"AMAZING!!!"  → compound 0.6884
```

Same word, stronger signal. For product reviews and support tickets — text
written by people who are not being careful — this matters more than it sounds.

## The failure mode you need to know about

This endpoint is lexicon-based. It scores words it knows and ignores words it
doesn't. It is not reading for meaning.

So this review, which any human reads as glowing:

```
"The delivery was fast and the product exceeded my expectations"
→ {"neg": 0.0, "neu": 1.0, "pos": 0.0, "compound": 0.0}
```

scores as perfectly neutral. Not slightly positive — *neutral*. None of "fast",
"exceeded", or "expectations" carries sentiment in the lexicon. The praise here
is entirely implied, and implication is exactly what a lexicon can't see.

The same trap in the other direction:

```
"Waited 45 minutes and the food arrived cold"
→ compound 0.0     (neutral)

"Waited 45 minutes and the food arrived cold and disgusting"
→ compound -0.5267 (negative)
```

One explicit word flips it. The first sentence is a bad review by any human
standard, and it scores as neutral.

**What to do about it.** Treat neutral as "no explicit sentiment found," not as
"the customer feels fine." In practice that means:

- Don't report a neutral bucket as satisfaction. It's mostly a *silence*
  bucket, and it will be large.
- Route neutral-scored reviews to a human, or to a stronger model, rather than
  closing them out. That's where your unhappy-but-polite customers are.
- Watch the *distribution* over time rather than individual scores. Lexicon
  methods are noisy per-review and much steadier in aggregate.

Where this approach genuinely shines: high volume, informal, emphatic text —
social posts, support chats, app store reviews — analysed in aggregate, with no
model to host and a few milliseconds per call.

## Scoring a batch efficiently

There's no bulk endpoint, so concurrency is how you go fast. Keep it modest and
reuse the connection:

```python
import requests
from concurrent.futures import ThreadPoolExecutor

session = requests.Session()

def score(review):
    r = session.get("https://use.textapi.com/sentiment", params={"text": review})
    r.raise_for_status()
    return review, r.json()["compound"]

with ThreadPoolExecutor(max_workers=8) as pool:
    results = list(pool.map(score, reviews))
```

Each call is one metered request, so a 10,000-review backfill is 10,000
requests — worth knowing before you point it at your whole history.

## Combining it with entities

The question behind "what's our sentiment" is usually "sentiment about *what*."
Split the review into sentences, score each, and run `/ner` alongside to find
what each sentence is about — that gets you "shipping: negative, product:
positive" from a single mixed review, which is far more actionable than one
number for the whole thing.
