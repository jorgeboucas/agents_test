---
name: Research Assistant
description: Research a question and publish a cited report
on:
  workflow_dispatch:
    inputs:
      question:
        description: Research or technical question to investigate
        required: true
        type: string
      depth:
        description: Level of detail
        required: true
        type: choice
        options:
          - brief
          - detailed
        default: brief

concurrency:
  job-discriminator: ${{ github.run_id }}

permissions:
  contents: read
  copilot-requests: write

engine: copilot

tools:
  bash:
    - "cat *"

network:
  allowed:
    - defaults
    - api.crossref.org
    - api.openalex.org

steps:
  - name: Fetch scholarly sources
    uses: actions/github-script@v9.0.0
    env:
      RESEARCH_QUESTION: ${{ github.event.inputs.question }}
    with:
      script: |
        const fs = require("fs");

        const question = process.env.RESEARCH_QUESTION;
        const searchQuery = question.replace(/[?*]/g, " ").replace(/\s+/g, " ").trim();
        const outputPath = "/tmp/gh-aw/agent/research-sources.json";

        const openAlexUrl = new URL("https://api.openalex.org/works");
        openAlexUrl.searchParams.set("search", searchQuery);
        openAlexUrl.searchParams.set("per-page", "15");
        openAlexUrl.searchParams.set(
          "select",
          "id,doi,title,publication_year,publication_date,type,authorships,primary_location,abstract_inverted_index,cited_by_count"
        );

        const crossrefUrl = new URL("https://api.crossref.org/works");
        crossrefUrl.searchParams.set("query.bibliographic", searchQuery);
        crossrefUrl.searchParams.set("rows", "15");
        crossrefUrl.searchParams.set(
          "select",
          "DOI,title,author,published,container-title,type,URL,abstract,is-referenced-by-count"
        );

        const fetchJson = async (name, url) => {
          const response = await fetch(url, {
            headers: { "User-Agent": "gh-aw-research-workshop/1.0" },
            signal: AbortSignal.timeout(30000),
          });
          if (!response.ok) {
            throw new Error(`${name} returned HTTP ${response.status}`);
          }
          return response.json();
        };

        const abstractFromIndex = (index) => {
          if (!index) return null;
          const words = [];
          for (const [word, positions] of Object.entries(index)) {
            for (const position of positions) words[position] = word;
          }
          return words.join(" ").slice(0, 1500);
        };

        const results = await Promise.allSettled([
          fetchJson("OpenAlex", openAlexUrl),
          fetchJson("Crossref", crossrefUrl),
        ]);

        const openAlex = results[0].status === "fulfilled"
          ? results[0].value.results.map((work) => ({
              title: work.title,
              doi: work.doi,
              url: work.id,
              publication_year: work.publication_year,
              publication_date: work.publication_date,
              type: work.type,
              venue: work.primary_location?.source?.display_name ?? null,
              authors: work.authorships.slice(0, 5).map((entry) => entry.author.display_name),
              cited_by_count: work.cited_by_count,
              abstract: abstractFromIndex(work.abstract_inverted_index),
            }))
          : [];

        const crossref = results[1].status === "fulfilled"
          ? results[1].value.message.items.map((work) => ({
              title: work.title?.[0] ?? null,
              doi: work.DOI ?? null,
              url: work.URL ?? null,
              published: work.published?.["date-parts"]?.[0] ?? null,
              type: work.type ?? null,
              venue: work["container-title"]?.[0] ?? null,
              authors: (work.author ?? []).slice(0, 5).map((author) =>
                [author.given, author.family].filter(Boolean).join(" ")
              ),
              cited_by_count: work["is-referenced-by-count"] ?? null,
              abstract: work.abstract?.replace(/<[^>]+>/g, " ").replace(/\s+/g, " ").trim().slice(0, 1500) ?? null,
            }))
          : [];

        const errors = results.flatMap((result, index) => {
          if (result.status !== "rejected") return [];
          const source = index === 0 ? "OpenAlex" : "Crossref";
          return [`${source}: ${result.reason.message}`];
        });

        if (openAlex.length === 0 && crossref.length === 0) {
          throw new Error(`No scholarly sources were retrieved. ${errors.join("; ")}`);
        }

        fs.mkdirSync("/tmp/gh-aw/agent", { recursive: true });
        fs.writeFileSync(
          outputPath,
          JSON.stringify({
            question,
            fetched_at: new Date().toISOString(),
            openalex: openAlex,
            crossref,
            retrieval_errors: errors,
          }, null, 2)
        );
        core.info(`Saved ${openAlex.length} OpenAlex and ${crossref.length} Crossref records`);

safe-outputs:
  allowed-domains:
    - doi.org
    - openalex.org
  create-issue:
    title-prefix: "[research] "
    expires: 30d
    max: 1

timeout-minutes: 15
max-ai-credits: 200
---

# Research Assistant

Research this question:

> ${{ github.event.inputs.question }}

Requested depth: **${{ github.event.inputs.depth }}**

## Research method

1. Read `/tmp/gh-aw/agent/research-sources.json`.
2. Compare and deduplicate the OpenAlex and Crossref records.
3. Prefer primary and peer-reviewed sources. Clearly label preprints or records whose review status cannot be verified.
4. Check publication dates and distinguish recent evidence from older background.
5. Cross-check important claims against more than one independent source when possible.
6. Clearly separate publication metadata, evidence from available abstracts, your synthesis, and unresolved uncertainty.
7. Never invent a citation, result, author, date, or link. If the retrieved records do not provide enough evidence, say so.

## Output

Create one GitHub issue containing:

- the original question
- a concise answer
- key findings with inline source links
- important disagreements or evidence gaps
- limitations of this search
- three useful follow-up questions

Use a descriptive issue title after the required `[research]` prefix.