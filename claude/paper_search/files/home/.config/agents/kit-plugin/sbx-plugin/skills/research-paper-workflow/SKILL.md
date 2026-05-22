---
name: research-paper-workflow
description: Rules and workflow for finding, retrieving, and analyzing papers using the kit's paperclip, semanticscholar, and deepwiki MCP tools. Use when the user asks about a paper, preprint, regulatory document, clinical trial, or any literature topic by author, title, DOI, PMC/PMID, arXiv ID, or NCT.
---

# Research-paper workflow

When the user asks anything about a paper, preprint, regulatory document, or clinical trial, prefer the `paperclip` MCP tool over `curl`, `WebFetch`, or downloading PDFs from publisher sites. Paperclip indexes the full text already — don't re-OCR what's already parsed.

## Tool roles

| Tool | Best for |
|---|---|
| **paperclip** | Full-text search, exact `lookup` by DOI/PMC/PMID/arXiv ID, `map` over result sets, section-by-section reading, SQL over metadata. Covers bioRxiv, medRxiv, PubMed Central, arXiv (all categories — CS, ML, physics, math too), OpenAlex abstracts, FDA, ClinicalTrials.gov, international regulatory + trials. Source list keeps growing — when unsure if a paper is indexed, try `lookup` first. |
| **semanticscholar** | Author search, citation graphs, broader-than-paperclip identifier discovery. Use it to find papers, then hand the PMC/DOI/arXiv ID to paperclip for the text. |
| **deepwiki** | Public GitHub repo Q&A. Use for tools, codebases, and methods referenced in papers (e.g. AlphaFold, ESM, Boltz, RFdiffusion). Not for paper content. |
| **WebFetch / curl** | Only when none of the above has the content: vendor docs, news, blog posts, patents, slide decks, conference talks not in arXiv/PMC, or a specific URL the user gave you. |

## The bridging pattern

When the user names an author or topic but you need the paper text:

1. **semanticscholar** — search by author/topic → collect identifiers (DOI, PMC, arXiv) from the results.
2. **paperclip `lookup`** by the most precise identifier:
   - `lookup doi 10.1038/...`
   - `lookup pmc PMC11402997`
   - `lookup arxiv 2403.03507`
   - `lookup pmid 32943797`
   Exact match — won't miss the way bare author-search can.
3. **paperclip read** — `cat /papers/<id>/meta.json` for metadata, `head` or section files for skimming, `map --from <result_id>` for parallel Q&A across multiple papers.

Anti-pattern to avoid: semanticscholar → `curl` from nature.com / `WebFetch` of a PMC HTML page → save PDF. The paper is already in paperclip indexed and parsed. Skip the download.

## When paperclip isn't the right tool

- Specific URL provided by the user that isn't on a paperclip-indexed source → WebFetch directly.
- Patents, slide decks, technical reports outside the indexed sources → WebFetch.
- Questions about a *codebase* (not a paper that uses it) → deepwiki.

If unsure whether paperclip indexes a source, try `lookup` or `search` first — it's cheap. If the result is empty, fall back to WebFetch.

## Identifier hygiene

- PMC IDs look like `PMC11402997` → `lookup pmc PMC11402997`.
- DOIs → `lookup doi 10.xxxx/...`.
- arXiv → `lookup arxiv 2403.03507`.
- PMID → `lookup pmid 32943797`.
- bioRxiv/medRxiv IDs come back from search prefixed `bio_` / `med_` — usually no separate lookup needed; cat the path directly.

## Multi-paper questions

Use `map --from <search_result_id>` to read N papers in parallel rather than `cat`-ing each one. Map runs an LLM reader on every paper and returns per-paper answers — much faster and lower context cost than sequential reads.

## Citations

When paperclip returns citation URLs (format `https://citations.gxl.ai/papers/<id>#L<line>`), cite findings inline with `[1]`, `[2]` markers and end with a REFERENCES footer. Never expose internal `doc_id` values in prose.
