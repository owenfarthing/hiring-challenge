# PLAN.md (commit this BEFORE reading CLARIFICATIONS.md or writing solution code)

> Delete the prompts below and replace with your own. Keep it tight.

## Architecture
I'd build a simple web application to accomplish this job - TypeScript, Node, React, Vite. Zod for validation, Express for routing since this is a very straightforward ask. Node server handles data processing and enrichment (take pressure off the client).
Frontend components (exluding basic components like cards, buttons, etc):
- CSV upload dialog
- Results table, populated from server
Data flow:
- Depending on CSV size (see clarifying questions), send or stream to backend for processing
- Parse out company names and use to query enrichment sources
- Evaluate enrichment results using confidence score logic
- Send results (and cache ID) to frontend for display

*Recommended future enhancement: Store contact capture results in a database or cache to avoid repeated file uploads when searching for the same contacts.
Components:
- "Recent searches" navigable list, from cache/db
Data flow:
- Store result in cache/db before sending to client, send added row/key with response for trivial state refresh

## Sources & strategy
Evaluation of sources:
- registry seems most likely to give us role info (most important)
- listing may provide phone number (sometimes unpopulated, not always the same individual returned from the registry)
- enrichment sometimes provides phone number/email, as well as a confidence score

Concerns:
- registry/listing/enrichment info does not always exist
- important fields are not always populated
- confidence score is often low on encrichment data
- different individuals may be listed under different sources
- enrichment data does not provide a name at all; this begs the question, can we ever verify that the enrichment data matches the registry/listing?

Strategy:
- combine registry and listing sources when "name" field aligns; if it doesn't, prioritize registry if it exists and includes role; otherwise prioritize listing since it may contain contact info
- if we can verify listing phone matches enrichment phone, combine; if we cannot verify either way, combine and indicate "needs human review"

## Quality
Deduping:
- dedupe input by company name
- if necessary, dedupe multiple ouput rows from sources by value / confidence score when applicable (what fields are populated, if there is confidence scoring, compare)

Confidence score logic (see clarifying questions on this):

Rules:
- registry role must fall into a valid category (non-empty, reasonable matching against requirement) to be considered full-confidence
- names must match between registry and listing to combine
- phone numbers must either match between listing and enrichment, or one must be unpopulated, to combine

Logic:
Given the above rules for combining/handling sources, allot a certain number of possible confidence points to each property:
- role, 49 (if we are not confident about the role, providing contact info is less valuable)
- phone, 30 (slightly higher than email because it represents a link between registry, listing, and enrichment)
- email, 20

Role scores 0 when empty, 10 when it doesn't match any expected role, 49 when populated and matching

Phone scores 0 when:
- unpopulated in listing and enrichment
- name in listing doesn't align with name in registry
Phone scores 10 when:
- unpopulated in listing but populated in enrichment
- and name in listing aligns with name in registry
Phone scores 20 when:
- populated in listing but populated differently in enrichment
- and name in listing aligns with name in registry
Phone scores 30 when:
- populated in listing AND enrichment
- and name in listing aligns with name in registry

Email scores 0 when:
- unpopulated in enrichment
- phone number in listing doesn't match phone number in enrichment
Email scores 10 when:
- populated in enrichment
- and phone number is unpopulated in listing or enrichment
Email scores 20 when:
- populated in enrichment
- and phone number matches between listing and enrichment

Show disclaimer in frontend to notify users that false positives are possible
Always leave remaining 1% of confidence score (highest possible score is 99%)

## Privacy / compliance
Do not send frontend any PII / personal data; only send the client the exact fields listed in requirements
Best-effort validation for outgoing data to ensure each field output matches expectation

## Clarifying questions

1. Is there a confidence threshold at which we consider an output to be non-verifiable?
   - Why it matters:
   Determines at what point the confidence in an output is too low to be referencable
   - Default assumption:
   Establish a reasonable threshold, say 75% (criticially, eliminates rows that have a populated but invalid role)
   - What changes if answered:
   The code will be generalizable to accept a configurable threshold, but we might need to revisit weighting logic if it changes; if no threshold at all, only handle "not found" as nonverifiable

2. What are our upper bounds on a) input CSV # of rows and b) source output # of rows?
   - Why it matters:
   Determines technical strategy on data ingestion and querying
   - Default assumption:
   Reasonable I/O bounds (input < 100 rows, okay to send without streaming; output ~1-3 rows per source per company - ~ 300 to 1k rows, okay to display without server-side pagination)
   - What changes if answered:
   If input file size pushes 50MB, switch to streaming (more complex ingestion process with progress reporting)
   Also consider paginating result, especially if output pushes server-side memory limits; requires database or other storage to hold full result + server-side pagination filter support

3. Should we expect to handle multiple rows per source per company?
   - Why it matters:
   Complicates confidence logic (how do we determine which of two results is more trustworthy/more valuable?)
   - Default assumption:
   Reasonable to assume one output row per source per company (judging by the enrichment_responses.json examples)
   - What changes if answered:
   Build prioritization logic
   Example for registry source:
   - compare populated fields - do both contain a role?
   - compare role values - are both roles valid? possibly implement role hierarchy/priority to prioritize higher roles
   - which row(s) match our other source data (e.g. listing names)
   - fall back to "pick first" strategy if we cannot differentiate, or consider returning multiple contacts per company