# Contributing to the LLM Wiki

The wiki is a curated, source-backed knowledge base. A page should teach one
idea clearly and remain useful even when the source repository changes.

## Workflow

1. Discover the source material and record its date, provenance, and confidence.
2. Classify each item before writing:
   - reusable concepts, methods, or source notes -> this wiki;
   - live contracts and design decisions -> beside the owning code;
   - stale architecture, plans, and historical reviews -> the source repo's
     `archived/` tree;
   - generated reports, campaign logs, and private working notes -> remain in
     the source repo or its archive.
3. Extract rather than copy. Make one focused page answer one question, with
   intuition first, mechanics second, and limitations visible.
4. Cite primary sources. Separate sourced claims, repository observations, and
   the author's own inference.
5. Add the page to the nearest index and link related concepts using relative
   wiki links. Do not make the wiki the live contract for another repository.
6. Review the diff for stale paths, unsupported claims, broken links, and
   accidental generated or private material.
7. Commit a small, topical change. Source-repository archive moves should be a
   separate commit from the wiki content when both are involved.

## Page shape

Prefer this structure unless the topic clearly needs another one:

```text
# Topic
## Intuition
## Mechanics
## Practical implications
## Limits and evidence
## Sources
```

## Checks

Before committing, run `git diff --check`, inspect every new link, and confirm
that the relevant index or glossary entry is updated. A future site builder or
link checker may automate these checks; until then they are part of review.
