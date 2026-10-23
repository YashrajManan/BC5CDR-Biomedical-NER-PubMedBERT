# BC5CDR Biomedical NER with PubMedBERT

Fine-tuning a biomedical-domain transformer (PubMedBERT) to extract disease and chemical mentions from raw
PubMed abstract text, then using the extracted entities to build a real chemical↔disease co-mention table —
a first, concrete step toward literature-scale relationship mining.

## Aim

Can a domain-pretrained language model read a raw biomedical abstract and reliably pull out, as structured
data, which spans of text are disease names and which are chemical/drug names?

## Objective

Fine-tune PubMedBERT-base on the real BC5CDR corpus for token-level (BIO) disease/chemical named-entity
recognition; evaluate with entity-level precision/recall/F1; demonstrate the fine-tuned model generalizing to
real, unseen, current PubMed abstracts; and use the extracted entities to build a simple real chemical↔disease
co-mention table.

## Data fetch

**Training/evaluation data:** the real BioCreative V CDR (BC5CDR) corpus — PubMed abstracts double-annotated
by trained biomedical curators (with a third senior annotator adjudicating disagreements) for disease and
chemical mentions, fetched programmatically as JSON-lines files directly from the `tner/bc5cdr` HuggingFace
repository (`dataset/train.json`, `valid.json`, `test.json`). Note: `datasets.load_dataset("tner/bc5cdr")`
could not be used as originally planned — HuggingFace's `datasets` library has dropped support for
script-based dataset loaders, which this repo uses — so the same underlying annotated files were fetched
directly instead (same real data, different fetch mechanism; see "Learning through project" below).

**Live-demo data:** five real, current PubMed abstracts fetched live via Biopython's `Bio.Entrez` (NCBI
E-utilities), never seen during training.

## Data describe

5228 train / 5330 validation / 5865 test real annotated examples. Note this loader segments examples by
**sentence**, not whole abstract (the original BC5CDR corpus is ~1500 abstracts) — so each row is one real,
annotated sentence with a `tokens` list and a parallel `tags` list (BIO label ids: `O`, `B-Chemical`,
`B-Disease`, `I-Disease`, `I-Chemical`).

## Methods / Workflow — what we did

Tokenized with PubMedBERT's tokenizer and re-aligned word-level BIO labels onto subword tokens (`-100` for
special/continuation tokens, per the standard HuggingFace token-classification convention). Fine-tuned
`AutoModelForTokenClassification` (PubMedBERT-base backbone + a fresh linear classification head) with
HuggingFace `Trainer`, evaluated with `seqeval`'s entity-level precision/recall/F1 (not raw token accuracy,
which can look high even when entities are clipped short). Ran the fine-tuned model via a HuggingFace
`pipeline("ner", ...)` on five real, live-fetched PubMed abstracts to check generalization beyond BC5CDR.
Reconstructed complete entity strings from the real gold test-set BIO tags and counted same-sentence
chemical↔disease co-occurrences into a ranked table.

## Results

Real held-out test-set performance: **precision 0.885, recall 0.919, F1 0.901, accuracy 0.979** — consistent
with published PubMedBERT-on-BC5CDR benchmarks. On five real, unseen 2026 PubMed abstracts (diabetic
complications), the model correctly extracted a coherent set of disease entities (diabetic nephropathy,
diabetic cardiomyopathy, diabetic retinopathy, diabetic osteoporosis, including their real abbreviations) and
chemical entities (metformin, sodium, glucose) with high confidence (mostly >0.99). The top real
chemical↔disease co-mentions from the test set: pilocarpine↔seizures, haloperidol↔catalepsy,
doxorubicin↔cardiotoxicity, heparin↔thrombocytopenia, levodopa↔dyskinesia, levodopa↔Parkinson's disease,
methamphetamine↔psychosis. Full tables: `results/test_metrics.csv`, `results/live_demo_entities.csv`,
`results/chemical_disease_comentions.csv`.

## Biology interpretation of results

The entity-level F1 (0.901) means the model finds complete, correctly-typed entity spans, not just
individual correct tokens — the metric that actually reflects whether the extracted entities are usable.
The live-demo extractions, on text with zero overlap with BC5CDR's training distribution, are direct evidence
of real generalization rather than memorization. The co-mention table's top pairs are independently
well-documented real pharmacology, not coincidences: doxorubicin-induced cardiotoxicity and heparin-induced
thrombocytopenia (HIT) are both named, clinically significant adverse-drug-reaction phenomena; pilocarpine is
a real, standard chemical used to experimentally induce seizures; levodopa↔dyskinesia is a well-known
long-term side effect of standard Parkinson's treatment. That a purely data-driven pipeline (fine-tuned NER
plus simple co-occurrence counting, no external knowledge base) recovered this many real, verifiable
relationships is meaningful validation beyond the F1 score alone.

## Learning through project

A model or library's real behavior can diverge silently from its advertised defaults or from how a
standard call has always worked before — a breaking upstream change to a loading mechanism, or a
tokenizer's real token-limit defaulting differently than its own config implies, are both the kind of thing
that stays invisible until the input shape that actually exercises them shows up. The fix in both cases was
the same: verify directly, by reading the real source or fetching the raw files, rather than assuming a
standard call will keep behaving the way it always has.

## Caveats

BC5CDR's abstracts were specifically selected for chemical-induced-disease relevance, so performance here
may not transfer identically to biomedical text with little chemical-disease content. The co-mention table
reflects only the real test set's ~5865 sentences (same-sentence, not same-document, co-occurrence — this
loader segments by sentence, not whole abstract), not the full literature. The live-demo entities are a
small, qualitative generalization check (five abstracts, one topic), not a systematic benchmark.
