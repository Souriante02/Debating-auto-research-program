# Debate Structural Retrieval

A prototype that finds past debate motions with the **same argument structure** rather than the **same topic**, built in n8n during the n8n AI Automations Hackathon (University of Sydney, September 2026).

## The problem

Debate teams build up years of research, case files and lines of argument, but almost none of it gets reused. Motions are filed and searched by topic, so when a new motion comes up, coaches only find past work on the same subject.

Yet the arguments that actually transfer are usually structural. A motion about zero-hour contracts and a motion about payday lending look unrelated, but both ask the same question: should we protect the party with less bargaining power by taking away a flexible option they currently choose? A coach who sees that link can reuse an entire case.

## How it works

The workflow has two stages.

**1. Build the library (offline).** Each past motion is reduced by an LLM to a topic-free *argument skeleton*:

| Field | What it captures |
|---|---|
| `abstract_core` | One sentence stating what is permitted or prohibited, the two competing values, and what is given up. Every concrete actor or domain word is replaced with a structural description such as "the party with less bargaining power" or "the institution". |
| `value_clash` | The two values in tension |
| `burdens` | What each side must prove |
| `stakeholders` | The parties involved, described structurally |
| `core_mechanism` | The causal chain in dispute |

The library is then indexed twice in an in-memory vector store using Mistral embeddings: once on the original motion text (topical index) and once on the abstract core (structural index).

**2. Query a new motion (live).** The new motion is sent to an LLM (`gpt-oss-20b` via Groq) with the same prompt to produce its abstract core. The workflow then searches both indexes and returns the top 4 matches from each in a side-by-side table, so the two kinds of retrieval can be compared directly.

```
Run Demo → Seeded Skeletons → Index (topical + structural)
                                      ↓
Query Motion → LLM: compute abstract core → Topical Search ┐
                                          → Structural Search ┴→ Side-by-side comparison
```

## Design decisions

- **Pre-computed library.** The 12 seed skeletons were extracted ahead of time with one model and one prompt, so building the library makes no LLM calls. A live demo therefore depends on a single model call, which keeps it fast and means it cannot fail halfway through on stage.
- **A strict abstraction test.** The prompt bans domain words (work, wages, schools, loans, media and so on) and includes a self-check: if a reader could guess the topic from the abstract core alone, it must be rewritten. Without this, embeddings simply cluster by topic again.
- **Two indexes, one table.** Keeping the topical search alongside the structural one makes the difference visible rather than asserted.
- **Tolerant parsing.** Model output is cleaned of code fences and stray text before JSON parsing, and parse errors are surfaced instead of breaking the run.

## Running it

1. Import `debate-structural-retrieval.json` into n8n.
2. Add credentials for **Mistral Cloud** (embeddings) and **Groq** (chat model).
3. Edit the motion in the **Query Motion** node if you want to test your own (default: *"This house would ban unpaid internships"*).
4. Click **Run Demo** and open the output of the **Comparison** node.

## Limitations

- The library holds 12 motions, enough to demonstrate the idea but not to evaluate it.
- Retrieval quality has not been measured; the side-by-side output is a qualitative comparison.
- The vector store is in memory, so the library is rebuilt on every run.

## Next steps

- Grow the library from real tournament archives and the team's own case notes.
- Evaluate retrieval with coaches by rating whether structural matches lead to reusable arguments.
- Show which arguments from a matched motion transfer and where the analogy breaks.
- Add a transcript review feature that labels each clash in a round (unanswered, partial, holding) to speed up post-round feedback.

<img width="726" height="438" alt="image" src="https://github.com/user-attachments/assets/3f9cc7b0-7f13-45c0-9634-67829eae6289" />

<img width="733" height="465" alt="image" src="https://github.com/user-attachments/assets/a227d0d4-0847-422d-9868-265f9f967bdc" />

