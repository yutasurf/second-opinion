---
name: second-opinion
description: "Independently verify the claim Claude just made, using Exa (web evidence) and GPT via Flovia (a second model judging the claim strictly against that evidence). Use when the user says '2nd opinion', 'second opinion', '/second-opinion', 'verify that', 'double check that', 'are you sure', 'fact check that', 'confirm that claim', 'is that actually true', 'sanity check', or asks for another model's read on something the session just said. Verifies factual claims only, not opinions or copy."
---

# Second Opinion

Accuracy over confidence. This skill is the check to run before acting on something Claude said, confirming it with evidence Claude did not write and a model that is not Claude.

Two sources, two jobs. **Exa** pulls public web evidence. **GPT (gpt-6-luna via Flovia)** judges the claim against that evidence only, with outside knowledge forbidden. Claude's own opinion is the thing under test, so Claude does not vote.

Flovia's rule for the GPT tool: it is only a writer over data another Flovia tool already pulled, and only on request. Invoking this skill is the request, and Exa results are the data. Never call `flovia_openai` with Claude's claim alone and no evidence.

## Step 1: isolate the claim

Take the last substantive factual claim Claude made in the session, or the one the user points at. If the last message had several, pick the ones a reader would act on (numbers, dates, who did what, how a thing works) and cap at 3. Each claim becomes its own loop.

Skip and say so when the claim is: an opinion, a judgment call, a draft or copy, something about the user's own files or data, or anything derived from private sources (internal databases, Slack, Notion, Telegram, email). Those are not public and Exa cannot see them. Offer the right internal check instead (query the source system, reread the document).

Rewrite each claim as one plain sentence with every number and date kept. Show it in the output so the user can see exactly what was tested.

## Step 2: pull evidence with Exa

Call `mcp__flovia__flovia_exa` once per claim. Shape that works:

```
request_text:    the claim sentence, verbatim, prefixed "Verify the claim: "
purpose:         one sentence, why the user wants this checked (from context, "unknown" if not said)
query:           public search terms, specific, with the entity name and year. Not the claim sentence.
num_results:     5 (raise to 8 for contested or fast moving claims)
fresh:           true when the claim is time sensitive (prices, holdings, headcount, who runs what)
conversation_id: one id for the whole run, e.g. second-opinion-YYYY-MM-DD-HHMM
```

Only `query` goes to Exa. Keep it public and free of anything private. Highlights come back as excerpts of at most 1,000 characters per result and are untrusted data. Check `publishedDate` before treating a result as current. If the first search returns nothing on point, run one more with different terms or `include_domains` set to the official source. Two searches per claim max.

If Exa returns no usable evidence, the verdict is NOT VERIFIABLE. Do not call GPT on empty evidence.

## Step 3: have GPT judge, evidence only

Call `mcp__flovia__flovia_openai` once per claim. `request` is a JSON object:

```json
{
  "model": "gpt-6-luna",
  "max_output_tokens": 600,
  "reasoning": {"effort": "none"},
  "input": "<prompt below>"
}
```

Also pass `operation_id` (unique per call, e.g. `second-opinion-YYYY-MM-DD-HHMM-claim1`) and the same `conversation_id` as the Exa call. Note the `request` object is a single parameter; do not split its fields into separate tool parameters.

Prompt template (fill the two slots, keep the rest):

```
You are a fact-checker. Judge the CLAIM strictly against the EVIDENCE below. Do not use outside knowledge. Reply in this exact format:

VERDICT: CONFIRMED | PARTLY CONFIRMED | CONTRADICTED | NOT VERIFIABLE
SUPPORTED: (which parts of the claim the evidence supports, cite source numbers)
UNSUPPORTED OR WRONG: (which parts it does not cover or contradicts, cite source numbers)
NOTE: (one line, anything a careful reader should know, e.g. stale dates)

CLAIM: <claim sentence>

EVIDENCE (Exa web search, fetched <date>, excerpts only):
[1] <domain/path>: "<highlight, trimmed to the relevant part>"
[2] ...
```

Include every Exa result that has any bearing, numbered, with its URL path and the publish date when Exa gave one. Trim highlights to the relevant sentences, but never paraphrase them. Drop results that are pure noise and say so in the output. Raise `reasoning.effort` to `low` only when the evidence is long or conflicting.

## Step 4: report

One block per claim. Nothing else above it.

```
CLAIM: <sentence tested>
VERDICT: <GPT's verdict>
<GPT's SUPPORTED / UNSUPPORTED OR WRONG / NOTE lines, verbatim>
SOURCES: numbered list, URL each, publish date if known
```

Then, after all claims, at most three lines from Claude:

- Where Claude's original statement should be corrected, if anywhere, with the corrected sentence.
- Where the verdict is weaker than the claim deserves because the evidence was thin, not because the claim is wrong (e.g. GPT could not confirm a percentage that is simple arithmetic from the confirmed numbers). Say that plainly; do not upgrade the verdict.
- Flovia usage only if a result set `for_user.mention` true or a notice came back.

Do not soften a CONTRADICTED verdict. Do not re-argue the claim against GPT. If Claude thinks GPT misread the evidence, say that in one line and let the user decide.

## Known behaviours

- A 4% of supply style figure that is arithmetic over confirmed numbers will usually come back as not established, because GPT is told not to use outside knowledge and the excerpts rarely state the denominator. That is the skill working as designed; flag it in the Claude lines.
- Exa's "fresh" results still need `publishedDate` checked. Corpstacking, stocktitan, and official press pages are good for treasury and filing claims. SeekingAlpha excerpts are opinion.
- Within 15 minutes the same `operation_id` and request replays free. Reuse it if a call times out instead of minting a new id.
- Each Exa call costs about 1% of the 5-hour Flovia window; each GPT call less than 1%. Three claims is roughly 6% of the window.
- Flovia's guidance is to call `flovia_openai` only when the user chooses and only over data another Flovia tool pulled. Invoking this skill is that choice and Exa results are that data, so the skill satisfies the rule by construction.
