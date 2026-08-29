---
layout: post
title: "Named entity recognition without training a model"
summary: "Pull people, companies, money, and dates out of raw text with one HTTP call — including the character offsets you need to highlight them."
description: "Extract named entities (people, organizations, money, dates, locations) from text with a single API call. Covers entity labels, character offsets for highlighting, and choosing between the small and large models."
reading_time: 6
---

Named entity recognition is the step where unstructured text becomes something
you can query: *which companies are mentioned in this filing, and how much
money?* Training a model for it is a project. Calling one is a line of code.

## One call, structured output

```bash
curl -G "https://use.textapi.com/ner" \
  --data-urlencode "text=Satya Nadella said Microsoft will invest \$10 billion in OpenAI next January"
```

```json
{
  "spans": [
    {"text": "Satya Nadella", "start": 0,  "end": 13, "label": "PERSON"},
    {"text": "Microsoft",     "start": 19, "end": 28, "label": "ORG"},
    {"text": "$10 billion",   "start": 41, "end": 52, "label": "MONEY"},
    {"text": "next January",  "start": 63, "end": 75, "label": "DATE"}
  ]
}
```

Four facts from one sentence, each with a type and a position.

## The offsets are the useful part

`start` and `end` are character indices into the exact string you sent, with
`end` exclusive — the same convention as Python slicing. That means
`text[span["start"]:span["end"]]` always reproduces `span["text"]`, which is
what makes highlighting reliable:

```python
import requests

text = "Satya Nadella said Microsoft will invest $10 billion in OpenAI next January"
spans = requests.get(
    "https://use.textapi.com/ner", params={"text": text}
).json()["spans"]

# Rebuild the string with <mark> tags around each entity.
out, cursor = [], 0
for span in sorted(spans, key=lambda s: s["start"]):
    out.append(text[cursor:span["start"]])
    out.append(f'<mark data-label="{span["label"]}">{text[span["start"]:span["end"]]}</mark>')
    cursor = span["end"]
out.append(text[cursor:])

print("".join(out))
```

Walk the spans in order and slice — never search for the entity text with
`str.replace`. If "Microsoft" appears three times, replace hits the wrong one.

## The labels you'll actually see

The common ones, in rough order of how often they show up in business text:

| Label | What it catches |
|---|---|
| `PERSON` | People, real or fictional |
| `ORG` | Companies, agencies, institutions |
| `GPE` | Countries, cities, states |
| `LOC` | Non-political locations — mountains, bodies of water |
| `DATE` | Absolute and relative dates, including "next January" |
| `MONEY` | Monetary values with units |
| `PRODUCT` | Objects, vehicles, foods — not services |
| `CARDINAL` | Numerals that don't fall under another type |

Also present: `TIME`, `PERCENT`, `QUANTITY`, `ORDINAL`, `EVENT`, `WORK_OF_ART`,
`LAW`, `LANGUAGE`, `NORP` (nationalities and political groups), and `FAC`
(buildings, airports, highways).

## Small model or large model

The default is the small model. Pass `model=en_core_web_lg` for the large one:

```bash
curl -G "https://use.textapi.com/ner" \
  --data-urlencode "text=Satya Nadella said Microsoft will invest \$10 billion in OpenAI next January" \
  --data-urlencode "model=en_core_web_lg"
```

On that sentence, the large model finds a fifth entity the small one misses
entirely — `OpenAI` — which is the general pattern: the large model has better
recall on names it hasn't seen in an obvious context.

It also labels `OpenAI` as `GPE` — a geopolitical entity — which is simply
wrong. That's the honest state of off-the-shelf NER: recall improves with
model size, but labels on unusual or newer proper nouns are unreliable in both
models. If your downstream logic branches on the label, either constrain
yourself to the types the model is confident about (`PERSON`, `DATE`, `MONEY`
are dependable) or treat `ORG`/`GPE` as one bucket and disambiguate yourself.

## Rendering it for a UI

If the end goal is showing highlighted text to a human rather than processing
spans in code, `/ner/display` skips a step and returns rendered markup:

```bash
curl "https://use.textapi.com/ner/display?text=Apple+hired+Jane+Doe+in+Paris&format=html"
```

`format=svg` (the default) returns an inline fragment you can drop into an
existing page; `format=html` returns a complete standalone document. You can
also pass your own `spans` as JSON — useful when you've corrected the model's
output, or when the entities come from your own database rather than from the
model at all.

## Practical notes

**Trim long documents.** Recognition on a 40,000-character article is slow, and
entities from paragraph 200 are rarely what you're after. The opening section
usually carries the ones that matter.

**Send real casing and punctuation.** These models learned on ordinary prose;
uppercasing everything or stripping punctuation measurably hurts accuracy.

**Deduplicate downstream, not upstream.** Every mention is returned separately,
including repeats. That's what you want for highlighting; for a summary of
"companies in this document," collapse on the entity text yourself.
