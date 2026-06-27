# Week 08: Text Cleaning and Fuzzy Matching

## 🎯 What you'll learn

Half the world's data is text in a column that pretends to be structured. **Product names, addresses, free-form comments, user-generated content** - none of it is clean. This week is the toolkit for taming it: Unicode normalization, regex patterns that actually work, fuzzy string distance, and lightweight named-entity extraction.

By the end of this week you'll be able to:

- Survive Unicode (NFC/NFD, mojibake, the BOM that ate Friday afternoon)
- Write regexes that are readable a year later
- Use **rapidfuzz** for string distance (Levenshtein, Jaro-Winkler, partial-ratio)
- Apply lightweight NER (spaCy / GLiNER) when you need entities pulled out of text fields
- Build a "string-to-canonical" pipeline for any messy text column

## 🧰 Lab setup

```bash
uv add rapidfuzz unidecode spacy regex
uv run python -m spacy download en_core_web_sm
```

## ✅ Your job

1. Read [theory.md](theory.md). Unicode is the part most engineers wing - don't.
2. Work through [lab.md](lab.md). Clean a real free-text column (NYC 311 service request descriptions) and pull out entities.
3. Build a fuzzy-match pipeline that links customer-typed addresses to canonical street names with ≥85% accuracy on a labeled sample.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Unicode for the Working Programmer](https://manishearth.github.io/blog/2017/01/14/stop-ascribing-meaning-to-unicode-code-points/) | The mental model | 30 min |
| [rapidfuzz docs](https://maxbachmann.github.io/RapidFuzz/) | The tool | 30 min |
| [spaCy 101](https://spacy.io/usage/spacy-101) | NER overview | 45 min |
| [Russ Cox - Regular Expression Matching Can Be Simple And Fast](https://swtch.com/~rsc/regexp/regexp1.html) | Why some regexes are slow | 30 min skim |

## 💡 What you should already know

- Weeks 01-07
- Regex basics - character classes, groups, anchors

---

**Next**: [Week 09: Joins at Scale →](../week-09-joins-at-scale/readme.md)
