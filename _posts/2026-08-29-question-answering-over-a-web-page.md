---
layout: post
title: "Ask questions about a web page in one API call"
summary: "Point the QA endpoint at a URL and a question, get an extracted answer with a confidence score — plus the two mistakes that make extractive QA go wrong."
description: "Use extractive question answering over any URL or text with a single API call. Covers the confidence score, why scoping the context matters, and how to handle answers the model invents confidence for."
reading_time: 7
---

Extractive question answering does something narrower than a chatbot, and the
narrowness is the point: it finds the span of *your* text that answers a
question. It can't wander off-source, because everything it returns is a
substring of what you gave it.

## Question plus context

```bash
curl -G "https://use.textapi.com/qa" \
  --data-urlencode "question=What is the refund window?" \
  --data-urlencode "context=Our return policy allows refunds within 30 days of delivery. Shipping fees are non-refundable."
```

```json
{"score": 0.3742, "start": 40, "end": 59, "answer": "30 days of delivery"}
```

`start` and `end` are character offsets into the context, so you can always
show the answer *in place* — highlighted in the source paragraph rather than
floating free. For a support-doc search, that context is often more valuable
to the reader than the answer string itself.

## Question plus URL

`POST /qa` accepts a `url` instead of `context` and does the article extraction
for you:

```bash
curl -X POST "https://use.textapi.com/qa" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "When was the first machine translation experiment?",
    "url": "https://en.wikipedia.org/wiki/Natural_language_processing"
  }'
```

One call: fetch, strip the page down to article text, answer against it. Which
brings us to the part most tutorials skip.

## Mistake one: a confident answer is not a correct answer

That exact call returns:

```json
{"score": 0.9059, "start": 42508, "end": 42512, "answer": "1989"}
```

A confidence of **0.91** — and the answer is wrong. The Wikipedia article says
the Georgetown experiment took place in **1954**, and says so about 900
characters in.

So where did 1989 come from? Offset 42508 is near the very end of a
42,596-character document, inside a bibliography entry:

> David M. W. Powers and Christopher C. R. Turk (1989). *Machine Learning of
> Natural Language.* Springer-Verlag.

The model found a year, in a line mentioning natural language, and scored it
highly. It had no way to know that a reference list is not prose.

The lesson isn't that the model is bad. It's that **the score measures how well
a span matches the question pattern, not whether the answer is true**. Score is
useful for ranking and thresholding; it is not a fact-check.

## Mistake two: feeding it the whole document

The root cause above is context length. Extractive QA degrades as context grows
— more candidate spans, more chances for a spurious match, and slower responses
(that 42,000-character call took over a minute).

Scope the context and the same question is answered correctly:

```python
import requests

article = requests.get(
    "https://use.textapi.com/text",
    params={"url": "https://en.wikipedia.org/wiki/Natural_language_processing"},
).json()["text"]

# Answer against the opening section, not the reference list.
answer = requests.post(
    "https://use.textapi.com/qa",
    json={
        "question": "When was the first machine translation experiment?",
        "context": article[:4000],
    },
).json()
```

The general rule: retrieve first, answer second. Find the handful of paragraphs
plausibly containing the answer — keyword search is usually enough — then ask
the question against those. This is faster, cheaper, and markedly more accurate
than one call over everything.

## There is no "I don't know"

The model always returns its best span, even when the context contains no
answer at all:

```
question: "Who is the CEO?"
context:  "Our return policy allows refunds within 30 days of delivery."

→ {"score": 0.1664, "answer": "Our return policy allows refunds within 30 days of delivery"}
```

It returned the entire sentence, with a score of 0.17. Nothing in the response
says "not found" — the low score is the only signal, and acting on it is your
job:

```python
MIN_SCORE = 0.30

result = requests.get("https://use.textapi.com/qa", params={
    "question": question, "context": context,
}).json()

if result["score"] < MIN_SCORE:
    answer = None          # show search results instead of a wrong answer
else:
    answer = result["answer"]
```

Calibrate that threshold on your own content rather than trusting the number
above — the right cutoff depends on how your documents are written. A useful
starting point: collect fifty real questions, look at the scores for answers
you know are right versus wrong, and put the line where they separate.

## When to reach for this

Good fits: FAQ and policy lookup, pulling a specific field out of
semi-structured documents, "find the sentence that says X" over a known corpus.
The answer is always traceable to a span in your source, which matters when
someone asks where a number came from.

Poor fits: questions whose answer requires combining facts from several
paragraphs, or any question whose answer isn't literally written in the text.
Extractive QA can only point at what's there.
