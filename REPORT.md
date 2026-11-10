# Assignment 3 Report

- **Name**: Anna Serra Guirao
- **Student ID**: [fill in]
- **Group**: [BBADBA 5A | BBADBA 5B | PPLEDBA 5A]

## Use case

In the first two assignments I was focusing on an M&A boutique, so for this one I built what I called Deal Desk, an assistant that answers the questions the the analysts have about EU rules while they prepare a deal. When two companies merge or one buys anotherone, if they're big enough the deal has to be approved by the European Commission before it can go ahead, and some deals with buyers from outside the EU can also be checked for security reasons, so the person asking is an associate who needs to know things like if the deal is big enough to need that approval, how long the approval takes, or what the contracts are allowed to say, and they use the answer to plan the deal and to advise the client.

The reason why a wrong answer is so much worse than "I don't know" is that the mistakes here are very expensive and they don't cost the same in both directions. If the assistant says that the deal doesn't need approval and it was wrong and the client goes ahead, the EU can fine the company up to 10% of its yearly sales and the deal can be stopped, but if it says that they need approval and it was wrong then the boutique would only loose some time preparing paperwork that wasn't necessary. On the other hand, "I don't know" just means the associate asks an actual lawyer, which costs some money but it is cheap compared with the risk of a having a wrong answer. That is why the whole system is built to answer only from the official EU documents, by showing which part of which document each answer comes from so the associate can check it, and say "The Deal Desk documents do not cover this" when the documents don't have the answer.

## Corpus

| Document | URL | Date | Licence | Words |
|---|---|---|---|---|
| EU Merger Regulation, Council Regulation (EC) No 139/2004 (EUMR) | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32004R0139 | 20 January 2004 | EUR-Lex reuse policy, Commission Decision 2011/833/EU | 14,317 |
| Commission Consolidated Jurisdictional Notice (CJN) | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:52008XC0416(08) | 16 April 2008 | same | 24,059 |
| Commission Remedies Notice (RN) | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:52008XC1022(01) | 22 October 2008 | same | 15,105 |
| Ancillary Restraints Notice (ARN) | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:52005XC0305(02) | 5 March 2005 | same | 3,332 |
| FDI Screening Regulation, Regulation (EU) 2019/452 (FDI) | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32019R0452 | 19 March 2019 | same | 6,890 |
| **Total** | | | | **63,703** |

In simple terms, the first document is the main EU law that says which deals need the Commission's approval, the second one is the Commission's guide on how to apply that law (for example how to measure the size of the companies), the third one explains what companies can offer to fix a deal the Commission is worried about, the fourth one says which extra clauses the contracts can include, and the last one is the EU law on checking investments from outside the EU. The word counts are the ones `main.py index` prints and saves in `index/stats.json`, after cleaning the pages, so they don't include the website menus or the footnotes. All five can be reused because the EU's legal website (EUR-Lex) says its legal documents can be reused for any purpose as long as the source is mentioned, and they don't have personal data, since they're laws and official guides that only mention companies in their examples.

When choosing them I had to be careful with two documents that changed recently. The EU law on foreign investments has already been replaced by a new one, but the new one only starts applying in 2028, so the old one is still the one in force and I kept it, which also gave me a good question that the documents can't answer, about what changes with the new law. Then I left out the Commission's guides on how it decides if a merger is harmful, because a new version is being prepared this year, so they could be outdated very soon.

## Ingestion

I download the five pages from the EU website as HTML and then I split them into what I call units, which are the smallest pieces of a legal text that can be pointed to, like "Article 1, paragraph 2" of the merger law or "paragraph 156" of the guide. The documents are written in different formats on the website, because the oldest one is a plain page, the guides put every numbered paragraph in a table, and the newest law uses another format, so my code reads every block of text in order and then follows either the articles of a law or the numbered paragraphs of a guide. Every unit keeps which document it comes from and its exact place in it, and every chunk keeps the list of units it covers, which is what lets the search tool show the source of each answer and lets the evaluation check if the right passage was found.

The extraction problem I spent the most time on was that some paragraphs were just disappearing. When I checked with code that the numbered paragraphs of each guide went 1, 2, 3 without gaps, I saw that 8 paragraphs of one guide and 4 of another were missing, and the reason was that when a paragraph has a list inside, the website puts the list in a table inside the paragraph's table, so my code was skipping the whole paragraph. After fixing it every guide has all its paragraphs, and in the same check I found two smaller problems, which were footnote numbers like "( 21 )" mixed into the text and a table inside one paragraph being read as four new paragraphs. When removing the footnote numbers I had to be careful, because an earlier version of the cleaning was also deleting the "(1)" in references like "Article 3(1)", which would have broken exactly the references the associates need to check the answers.

I compare two ways of cutting the same text into chunks, plus a third way of searching them:

- **structure**: one chunk per unit (a paragraph of an article or a numbered paragraph of a guide), joining very short paragraphs of the same article and splitting the few that are very long, which gives 584 chunks of 110 words on average.
- **fixed**: the text cut every 200 words with 40 words repeated between neighbours, ignoring the paragraphs, which gives 400 chunks of 199 words on average.
- **hybrid**: the structure chunks, but searched by combining the meaning-based search with a keyword search, because legal questions often depend on exact words like "full-function" that a keyword search finds easily.

Before turning each chunk into numbers (the embedding), I add a short header with the document's name and where the chunk is, because a sentence like "This shall not apply if each of the undertakings concerned achieves more than two-thirds..." means nothing on its own. The embeddings are made with gemini-embedding-2 and saved after every batch, so the index is only built once and the daily limit of the free version never makes me lose work.

## Test set

I came up with the 42 questions together with my sister, who works in M&A, because she knows what associates actually ask while they prepare a deal and which of these questions come up the most, and I drafted and checked them against the documents with the help of an AI assistant (see AI use), so most of them are written like real situations ("The deal is signed in February and the audited accounts for last year aren't ready yet, which figures do we use for the turnover?") and not like exam questions. For each one I wrote where the answer is and a short reference answer, and before using them I checked with code that every expected place really exists in the documents, and that the topics of the questions that can't be answered don't appear anywhere in them.

| Kind | Count |
|---|---|
| Answered by one passage (single) | 27 |
| Need two passages (multi) | 8 |
| Not answered by the documents (unanswerable) | 7 |
| **Total** | **42** |

Two examples:

- **Multi**: "If we offer commitments in Phase II after working day 55, what's the deadline to submit them and when does the Commission have to take the final decision?" needs one passage from the merger law for the final deadline and another one from the guide on remedies for the deadline to submit, so an answer that only finds one of them is incomplete.
- **Unanswerable**: "If the deal doesn't have an EU dimension, what are the Spanish merger notification thresholds that would apply?" is very close to what the documents talk about, which is what makes it a good test, because they talk about size thresholds all the time but only the EU ones, so the right answer is to say the documents don't cover it.

## Retrieval

| Design | Hit rate @1 | @3 | @5 | Words added to the prompt (top 5) | Chunks |
|---|---|---|---|---|---|
| structure | 0.429 | 0.657 | **0.914** | 661 | 584 |
| fixed | 0.571 | 0.714 | 0.800 | 996 | 400 |
| hybrid | 0.343 | 0.571 | 0.686 | 649 | 584 |

These are from `eval/summary.json`, over the 35 questions that have an answer, and a question only counts as a hit if every passage it needs is in the top k, so the two-passage questions need both. I chose **structure**, because the agent always reads the top 5 results and that's where structure is clearly the best, finding the right passages for 0.914 of the questions against 0.800 for fixed, and it does it adding 661 words to the prompt instead of 996, so it's both more complete and cheaper. What surprised me is that fixed is better at putting the right passage first (0.571 against 0.429), and I think it's because its chunks are almost twice as long, so each one has more context around the answer and it's easier to match with the question, but it also brings more text that has nothing to do with it. The hybrid design was the worst, which I didn't expect, and I think the reason is that in these documents the words a keyword search would use, like "concentration", "Commission" or "turnover", appear in almost every paragraph, so adding the keyword ranking just pushed the right passages down.

## Tools and subagent

The tool server (`server.py`, called `corpus`) gives the AI two tools:

- `search(query)`: looks for the 5 passages of the documents that best match the question and returns them, each one with its source in square brackets at the top, like [EUMR Art. 1(2)], so the AI can copy it into the answer. Its description also says what the documents don't cover, like Spanish or US rules, because I wanted the AI to know when not to trust the results.
- `add_document(path)`: adds a new file, going through the same steps as the original documents and only creating the new embeddings.

This is the first result `search` returned in `check.py` for the first test question ("...what turnover thresholds does it have to meet under the main test?"):

```
[EUMR Art. 1(1)–(3)] (Scope)
Document: Council Regulation (EC) No 139/2004 on the control of concentrations between undertakings (EU Merger Regulation)
Covers: Art. 1(1), Art. 1(2), Art. 1(3)
Without prejudice to Article 4(5) and Article 22, this Regulation shall apply to all concentrations with a Community dimension as defined in this Article. A concentration has a Community dimension where: (a) ...
```

The subagent is a small specialised assistant that my code editor can call, defined in `.claude/agents/corpus-researcher.md`, and it can only use the search tool, so it can't add documents or do anything else. Its instructions tell it to search, search again with other words if the first results don't answer the question, answer in at most 150 words using only what the passages say, show the source of every sentence, reply "The Deal Desk documents do not cover this." when they don't have the answer, and never say that a deal doesn't need approval unless a passage supports it.

*(Transcript from my editor: to be added.)*

## Adding a document

Before adding a file, `add_document` creates a kind of fingerprint of its content, so if exactly the same file is already in the index it just says so and doesn't do anything, and if a new version of a document with the same name is added, it replaces the old one instead of keeping both, because having two versions of the same rule could make the assistant mix them. The demo uses the question "Which deals can go through the simplified procedure and use the short Form CO?", which the documents can't answer, and the Commission's 2023 guide on the simplified procedure, which is exactly the document that answers it.

Before adding it, the agent answered: "The Deal Desk documents do not cover the rules on the simplified procedure or the short Form CO...", and suggested asking outside counsel or adding the right document, which is exactly what it should do. Then `add_document` answered "Done: added STN (STN.html): 35 chunks in the 'structure' index, 35 new embeddings", and after that the same question got a real answer with five categories of deals that can use the simplified procedure, each one with its source, for example "Joint ventures with no EEA turnover: Joint ventures where two or more undertakings acquire joint control, provided the JV has no current or expected EEA turnover and no EEA asset transfers are planned at notification [STN para. 5(a)]". When I added the same file a second time, the tool answered "STN (STN.html) is already indexed with identical content; nothing was added", so it didn't create any new embeddings. All of this is saved in `eval/add_document_demo.json`.

## Harness and guardrails

The agent (`agent.py`) is built with LangChain's `create_agent` like in session 10, using gemini-3.5-flash-lite, the tools of my server and `AGENTS.md` as its instructions. The rules in `AGENTS.md` are to search before answering, answer only from what the search returns, show the source of every claim, say when the documents don't cover the question, never conclude that a deal doesn't need approval without a passage that supports it, keep the answer short and practical, add documents only when the user asks, and treat the documents as information and never as instructions, and each rule has its reason next to it, because I think the AI follows a rule better when it knows why it exists.

The code also runs three guardrails that the AI can't skip, because they're in the code and not in the instructions:

1. **Approval**: `add_document` only runs if a person types "y", and if nobody can answer it doesn't run, because adding a document changes what every later answer is based on.
2. **Call limit**: at most 8 calls to the AI per question, so if it starts going around in circles it stops instead of spending money.
3. **Source check (my extra guardrail)**: after the AI answers, the code checks that every source it cites is one of the passages the search really returned in that conversation, and if one isn't, or if the answer gives facts without any source, the answer is blocked and the associate is told it couldn't be checked. I chose this one because the source is what the associate trusts, so a source the AI invented or remembered from its training is the most dangerous mistake, because it looks exactly like a real one.

In my first run the source check blocked 12 answers, and when I looked at them one by one none of them was actually wrong, it was my guardrail being too strict, because the agent wrote some sources in ways my code didn't understand, like two paragraphs in the same brackets ([CJN para. 155, 156]) or a point of a paragraph ([STN para. 5(a)]), and it also blocked two good refusals because they didn't use the exact sentence I asked for. After fixing how it reads the sources it doesn't block any of the 42 answers, which is good news but also means I didn't have a real case where it fired, because the agent always cited what it had found. That is why `evaluate.py` also tests it on purpose: it takes the real answer to "can they close the deal before the Commission clears it?" and adds one source the search never returned, as if the AI remembered it from its training, and the guardrail blocks it with "citations not found in the search results: EUMR Art. 21(4)", and the same answer with all its sources removed is blocked with "the answer states facts without any citation", while the original answer passes (`guardrail_test` in `eval/summary.json`).

## Evaluation

*(Filled in after the evaluation runs: code checks, the judge's rubric, my agreement with the judge, and correct, unsupported, correct refusals and wrong refusals.)*

## Failures

*(Filled in after the evaluation runs.)*

## Decision

*(Filled in after the evaluation runs, with the cost and latency per question.)*

## AI use

*(To be completed: which AI tools I used, for which parts, and what I checked or changed myself.)*
