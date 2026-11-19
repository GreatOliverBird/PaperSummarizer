# Scientific Paper Summary (80/20)

An extractive scientific-paper summarization skill that applies an 80/20-style selection process to a complete paper. It aims to retain the paper's most information-dense sentences while keeping the summary at or below 20% of the paper's eligible body-word count.

The summary is **verbatim**: selected sentences are copied from the source in their original order. The skill does not paraphrase, explain, or add connective text.

## What it does

Given a scientific paper as a PDF or a public link, the skill:

1. Extracts the paper's full text.
2. Identifies top-level chapters and merges their subsections into their parent chapters.
3. Calculates a mathematical importance weight for every chapter.
4. Scores eligible sentences using relevance, key-concept overlap, evidence/claim cues, and chapter position.
5. Selects high-scoring sentences under a 20% word budget, penalizing redundancy.
6. Restores the selected sentences to their original paper order.
7. Verifies that the selected sentences are verbatim source text and reports key-concept coverage.

## Input

- A local scientific-paper PDF, or
- A publicly accessible paper URL.

The complete paper must be extracted before any scoring begins. If the PDF text layer is empty, OCR is required. Two-column papers may need layout-aware extraction to prevent sentences from being interleaved.

## Selection rules

### Verbatim extraction

Every sentence in the summary must occur exactly in the source text. The only permitted cleanup is PDF-layout normalization, such as joining words split by line-end hyphenation, collapsing whitespace, and removing running headers, footers, or page numbers.

Selected sentences retain their original order. The summary must not contain paraphrases, editorial transitions, ellipses, inserted headings, or commentary.

### Word budget

The target budget is 20% of eligible body words. References, acknowledgements, funding statements, conflict disclosures, author contributions, data-availability statements, and ethics statements are excluded from the body-word budget. Excluded chapters still appear in the chapter table with a weight of zero.

The selector does not pad the summary to reach the budget and never selects a sentence that would exceed it.

### Sentence eligibility

The process excludes:
- Sentences shorter than six words.
- Figure and table captions beginning with `Fig`, `Figure`, or `Table`.
- Sentences in chapters assigned a zero role prior.

## Weighting and scoring

### Chapter weight

Each chapter is evaluated using four signals:

| Signal | Meaning |
|---|---|
| `C` | Cosine similarity between the chapter TF-IDF vector and the whole-paper TF-IDF vector. |
| `R` | A role prior based on the chapter heading. |
| `Q` | Claim-cue and numeric-token density relative to chapter word count. |
| `L` | Square root of chapter word count, giving chapter size diminishing returns. |

`C`, `Q`, and `L` are min-max normalized across chapters. The chapter score is:

```text
W_s = 0.30 * C_hat
    + 0.35 * R
    + 0.20 * Q_hat
    + 0.15 * L_hat

weight_s (%) = 100 * W_s / sum(W)
```

Role priors:

| Chapter type | Prior |
|---|---:|
| Abstract | 1.00 |
| Conclusion | 0.90 |
| Results / Experiments / Evaluation | 0.85 |
| Discussion / Limitations | 0.70 |
| Introduction / Background | 0.60 |
| Methods / Approach / Model | 0.55 |
| Unmatched heading | 0.50 |
| Related Work | 0.30 |
| Appendix | 0.15 |
| References and other excluded back matter | 0.00 |

### Sentence score

The whole paper is represented using TF-IDF. Its 40 highest-mass terms form the key-concept set, `K`.

For each eligible sentence `i` in chapter `s`:

```text
S_i = (W_s / max(W)) *
      (0.50 * c_i
     + 0.25 * k_i
     + 0.15 * q_i
     + 0.10 * p_i)
```

Where:

- `c_i` is the sentence's cosine similarity to the whole-paper TF-IDF vector.
- `k_i` is the share of the sentence's distinct terms that belong to `K`.
- `q_i` measures claim cues and numeric content, capped at 1.
- `p_i` is 1 for the first or last sentence of a chapter and 0 otherwise.

Claim cues include expressions such as `we show`, `we propose`, `we find`, `results indicate`, `outperform`, `significant`, `novel`, and `our method`.

### Redundancy-aware selection

The selector uses a greedy Maximum Marginal Relevance-style penalty with `lambda = 0.3`.

At each iteration, it chooses the highest-scoring sentence that still fits the remaining word budget. A candidate's score is reduced according to its maximum Jaccard similarity with already selected sentences:

```text
adjusted_score = S_i * (1 - 0.3 * max_jaccard_similarity)
```

Selection stops when no remaining candidate fits. The selected sentences are then sorted back into their original paper order.

## Output format

The skill returns three clearly separated parts:

### 1. Summary

One block containing only the selected verbatim sentences, in original paper order. No headings or commentary are inserted into this block.

### 2. Chapters and weights

A table listing every detected chapter:

| Column | Description |
|---|---|
| `#` | Chapter order |
| `Chapter` | Chapter title as it appears in the paper |
| `Words` | Chapter word count |
| `Weight %` | Normalized chapter importance |
| `Sentences kept` | Number of selected sentences from that chapter |

Chapter weights sum to 100%, including zero-weight excluded chapters.

### 3. Metrics

A metrics line reports:
- Eligible body-word count.
- Summary-word count and percentage of body words.
- Key-concept coverage.
- Extraction source (for example, local PDF or public URL).

Key-concept coverage is calculated as the proportion of the total TF-IDF mass of the 40 key concepts represented in the selected summary. If coverage is below 80%, the skill reports that result honestly; it does not add sentences merely to raise the metric.

## Verification

Before returning the result, the skill checks that:

1. Every selected sentence is a substring of the whitespace-normalized source text.
2. The summary does not exceed 20% of eligible body words.
3. The chapter weights are normalized to 100%.
4. The paper was read in full, or any extraction limitation is explicitly disclosed.

## Reference implementation

The skill includes a Python reference implementation using the standard library. It can be run against an extracted text file:

```bash
python3 -I script.py paper.txt
```

For a local PDF, extract text first, for example:

```bash
pdftotext -layout paper.pdf paper.txt
```

If extraction produces incorrect chapter boundaries—for example, because the paper uses unnumbered headings—the boundaries should be checked and corrected manually. The scoring equations remain unchanged.

## Important interpretation note

The 80/20 framing is an operational target, not a guarantee that exactly 80% of a paper's meaning has been preserved. The reported coverage metric measures representation of the algorithmically selected key concepts; it is a proxy and should not be interpreted as a validated measure of scientific understanding or completeness.
