# Assignment 3 Report

- **Name**: Anna Serra Guirao
- **Student ID**: [fill in]
- **Group**: [BBADBA 5A | BBADBA 5B | PPLEDBA 5A]

## Use case

In the first two assignments I worked for the same small M&A advisory boutique, first helping it find sell-side targets and then checking the company seal on contract pages in a buy-side data room, so for this one I stayed with them and built Deal Desk, an assistant that answers the associates' questions about EU merger control and EU foreign investment screening while they're preparing a deal. The person asking is an associate who has to know things like if the deal has to be notified to the European Commission, which turnover counts for the thresholds, if the client can close before clearance, or if the non-compete in the share purchase agreement is too long, and what they do with the answer is decide if the deal needs a filing, plan the timeline and draft the deal documents.

The reason why a wrong answer is so much worse than "I don't know" is that the mistakes here are very expensive and they're not symmetric. If the assistant wrongly says that no filing is needed and the client closes, the Commission can fine up to 10% of the group's turnover for closing without notifying and the deal can be blocked or delayed, while if it wrongly says a filing is needed the boutique only loses some time preparing a notification that wasn't necessary. On the other hand, "I don't know" just means the associate asks outside counsel, which costs around an hour of a lawyer's time, so it's a cheap answer compared with the risk of a confident wrong one. That is why the whole system is built to answer only from the legal texts, cite the exact article or paragraph so the associate can check it, and say "The Deal Desk documents do not cover this" whenever the documents don't have the answer.

## Corpus

| Document | URL | Date | Licence | Words |
|---|---|---|---|---|
| EU Merger Regulation, Council Regulation (EC) No 139/2004 (EUMR) | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32004R0139 | 20 January 2004 | EUR-Lex reuse policy, Commission Decision 2011/833/EU | 14,317 |
| Commission Consolidated Jurisdictional Notice (CJN) | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:52008XC0416(08) | 16 April 2008 | same | 24,059 |
| Commission Remedies Notice (RN) | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:52008XC1022(01) | 22 October 2008 | same | 15,105 |
| Ancillary Restraints Notice (ARN) | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:52005XC0305(02) | 5 March 2005 | same | 3,332 |
| FDI Screening Regulation, Regulation (EU) 2019/452 (FDI) | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32019R0452 | 19 March 2019 | same | 6,890 |
| **Total** | | | | **63,703** |

The word counts are the ones `main.py index` prints and saves in `index/stats.json`, after cleaning, so they don't include the EUR-Lex menus or the footnotes. All five texts can be reused because EUR-Lex's legal notice says that, unless stated otherwise, the legal documents it publishes can be reused for commercial or non-commercial purposes as long as the source is acknowledged, and they don't contain personal data, since they're legislation and Commission notices that only name companies in their examples.

When choosing the documents I had to be careful with two things that changed recently. The FDI Screening Regulation 2019/452 has already been replaced by a new regulation, (EU) 2026/1386, but the new one only applies from 17 January 2028, so 2019/452 is still the rule in force and I kept it, which also gave me a good unanswerable question about what changes with the new one. Then I left out the Commission's merger assessment guidelines from 2004 and 2008 on purpose, because the Commission published a draft to replace them in April 2026 and is planning to adopt them before the end of the year, so they could be outdated by the time someone uses this.

## Ingestion

I download the five pages from EUR-Lex as HTML (`index.py`, `download_corpus`), and then I extract them into what I call units, which are the smallest pieces of a legal text that have their own citation: a recital, a paragraph of an article like EUMR Art. 1(2), or a numbered paragraph of a notice like CJN para. 156. The regulations and the notices are written in different layouts on EUR-Lex, because the EUMR is an old page with plain paragraphs, the notices put every numbered paragraph in a table row, and the FDI Regulation uses the newer Official Journal format, so the extraction reads every block in order and then uses two parsers, one that follows "Article N" and its numbered paragraphs, and one that follows the numbered paragraphs of a notice together with the section heading they're under. Every unit keeps its source, its location and its heading, and every chunk keeps the list of units it covers, which is what lets the search tool show the exact citation and lets `evaluate.py` check if the right paragraph was found.

The extraction problem I spent the most time on was that some paragraphs were just disappearing. When I checked that the numbered paragraphs of each notice went 1, 2, 3 without gaps, I saw that 8 paragraphs of the Jurisdictional Notice and 4 of the Remedies Notice were missing, and the reason was that when a paragraph contains a list, EUR-Lex puts the list in a table inside the table row of the paragraph, so my code was skipping the outer row and losing the paragraph number and its first sentence. After fixing it, every notice has all its paragraphs, and in the same check I also found two smaller problems, which were that the footnote markers like "( 21 )" were mixed into the text and that the rows of a table inside a paragraph were being read as new paragraphs 1 to 4. I removed only the footnote markers with spaces inside the brackets, because legal references like "Article 3(1)" also have numbers in brackets and an earlier version of the cleaning was deleting them too, which would have broken exactly the citations an associate needs.

I compare two ways of chunking the same units, and a third design that reuses the first one's embeddings:

- **structure**: one chunk per legal unit, merging very short paragraphs of the same article (up to 300 words) and splitting the few that are longer than 300 words, which gives 584 chunks of 110 words on average.
- **fixed**: the document read as one stream and cut every 200 words with a 40-word overlap, ignoring the structure, which gives 400 chunks of 199 words on average, and its citation is the range of units it covers.
- **hybrid**: the structure chunks, but ranking them by combining the embedding search with a keyword search (BM25), because legal questions often depend on exact terms like "full-function" or "Article 22" that a keyword search finds well.

Before embedding, every chunk gets a short header with the document title, the location and the heading, because a paragraph like "This shall not apply if each of the undertakings concerned achieves more than two-thirds..." means nothing without knowing it's the thresholds article of the Merger Regulation. The embeddings are made with gemini-embedding-2 at 768 dimensions and saved in `index/vectors.jsonl` after every batch, so the index is only built once and the daily limit of the free tier never makes me lose an embedding.

## Test set

I came up with the 42 questions together with my sister, who works in M&A, because she knows what associates actually ask while they prepare a deal and which of these questions come up the most, and I drafted and checked them against the documents with the help of an AI assistant (see AI use), so most of them are written like real situations ("The deal is signed in February and the audited accounts for last year aren't ready yet, which figures do we use for the turnover?") and not like exam questions. For each one I wrote where the answer is and a short reference answer, and before using them I checked with code that every expected location really exists in the extracted units, and that none of the topics of the unanswerable questions appear anywhere in the corpus.

| Kind | Count |
|---|---|
| Answered by one passage (single) | 27 |
| Need two passages (multi) | 8 |
| Not answered by the corpus (unanswerable) | 7 |
| **Total** | **42** |

Two examples:

- **Multi**: "If we offer commitments in Phase II after working day 55, what's the deadline to submit them and when does the Commission have to take the final decision?" needs EUMR Art. 10(3) for the extension of the deadline and RN para. 88 for the 65 working days to submit commitments, and an answer that only finds one of them is incomplete.
- **Unanswerable**: "If the deal doesn't have an EU dimension, what are the Spanish merger notification thresholds that would apply?" is very close to the topic of the corpus, which is what makes it a good test, because the documents talk about thresholds all the time but only the EU ones, so the right answer is that the documents don't cover it.

## Retrieval

*(Filled in after `main.py evaluate`: hit rate at 1, 3 and 5 for each design, the words each one adds to the prompt and the design I chose.)*

## Tools and subagent

The MCP server (`server.py`, name `corpus`) has two tools:

- `search(query)`: searches the five documents and returns the 5 most relevant passages, each one with its citation in square brackets at the top, like [EUMR Art. 1(2)] or [CJN para. 156], the document's title and the units it covers. Its description says what the library covers and also what it doesn't, which is national or non-EU rules, because I wanted the model to know when not to trust the results.
- `add_document(path)`: runs a new file through the same extraction and chunking as the corpus, embeds only the new chunks and adds them to the index, and the file name becomes the short name used in citations.

*(One real search result is added here after the evaluation runs.)*

The subagent is defined in `.claude/agents/corpus-researcher.md` and can only use `mcp__corpus__search`, so it can't add documents or do anything else. Its instructions tell it to search, search again with other words if the first results don't answer the question, answer in at most 150 words only with facts from the passages, cite every sentence, reply "The Deal Desk documents do not cover this." when they don't have the answer, and never say that a deal doesn't need to be notified unless a passage supports it.

*(Transcript from my editor: to be added.)*

## Adding a document

`add_document` hashes the file before doing anything, so if exactly the same file is already in the index it answers that it's already indexed and doesn't embed anything, and if a file with the same short name but different content is added, it replaces the old version instead of keeping both, because in law having two versions of the same rule in the index is worse than having none. The demo uses the unanswerable question "Which deals can go through the simplified procedure and use the short Form CO?" and the Commission Notice on simplified treatment of 2023 (CELEX 52023XC0505(01)), which is exactly the document that answers it.

*(Answer before and after: filled in after the evaluation runs.)*

## Harness and guardrails

The agent (`agent.py`) is LangChain's `create_agent` with gemini-3.5-flash-lite, the tools of my MCP server and `AGENTS.md` as the system prompt, built from the harness of session 10. The rules in `AGENTS.md` are to search before answering, answer only from the search results, cite every claim with the exact citation of the passage, say when the documents don't cover the question, never conclude that no filing is needed without a passage that supports it, keep the answer short and practical, add documents only when the user asks, and treat the documents as data and not as instructions, and each rule has its reason written next to it, because I think the model follows a rule better when it knows why it exists.

The harness runs three guardrails that the model can't skip, because they're in the code and not in the prompt:

1. **Approval**: `add_document` only runs if a person types "y", and if nobody can answer it's refused, because adding a document changes what every later answer is based on.
2. **Call limit**: at most 8 model calls per question, so a question that makes the agent go around in circles stops instead of spending tokens.
3. **Citation check (my extra guardrail)**: after the model answers, the harness compares every citation in the answer with the passages the search tool actually returned in that conversation, and if a citation doesn't match any of them, or if the answer gives facts without any citation, the answer is withheld and the associate sees that it couldn't be verified. I chose this one because in this use case the citation is what the associate trusts, so a citation the model made up or remembered from training is the most dangerous kind of mistake, because it looks exactly like a correct one.

*(One case where it fired: filled in after the evaluation runs.)*

## Evaluation

*(Filled in after the evaluation runs: code checks, the judge's rubric, my agreement with the judge, and correct, unsupported, correct refusals and wrong refusals.)*

## Failures

*(Filled in after the evaluation runs.)*

## Decision

*(Filled in after the evaluation runs, with the cost and latency per question.)*

## AI use

*(To be completed: which AI tools I used, for which parts, and what I checked or changed myself.)*
