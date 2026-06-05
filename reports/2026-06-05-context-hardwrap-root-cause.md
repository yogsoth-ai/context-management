# Hard-Wrap Root-Cause Diagnosis

> Created: 2026-06-05
> Subject: Why context-checkpoint / context-init files came out with hard-wrapped prose
> Verdict: H1 (Claude Code's global hard-wrap habit) is the root cause. H2 is an independent real bug. H3 and H4 are rejected.

## Problem

Context files produced by the `context-init` and `context-checkpoint` skills contained hard-wrapped prose: every paragraph was broken into multiple physical lines at roughly 80–90 display columns, with a real newline ending each line rather than one continuous paragraph. The desired behavior is one paragraph written as one continuous line, with newlines used only to separate paragraphs, list items, headings, and fenced code blocks.

## Method

Four hypotheses were tested against evidence collected from the live `context/` directory and from the two skill definitions. Each hypothesis was assigned a verdict backed by a concrete, reproducible observation rather than intuition.

## H1 — Claude Code's global hard-wrap habit (CONFIRMED root cause)

Claude Code hard-wraps markdown prose at roughly 80 columns as a built-in writing habit, independent of these skills. Two pieces of evidence are decisive. First, spec files authored by entirely unrelated workflows — the `failure-enum-eval` and `vibe-researching-eval` specs produced through brainstorming and writing-specs, which never invoke the context skills — wrap exactly as heavily as the context files do (wrap-band counts of 196/371 and 174/387 non-blank lines sitting in the 78–92 column band). If the context skills were the cause, these unrelated files would not wrap; they do, so the cause is upstream of any skill. Second, a CJK-robust display-width measurement (counting CJK characters as two columns) showed every single context file averaging 65–76 columns with maximum line widths clustered at 86–95. There was no clean unwrapped exception once the measurement accounted for Chinese text — including the one file that a naive byte-based check had falsely flagged as unwrapped, which on correct measurement averaged 65 columns with zero lines over 100 columns, i.e. it was wrapped just like the rest.

## H2 — the ≥500-line constraint induced wrapping (REJECTED as cause, but a separate real bug)

The hypothesis was that authors wrap prose to inflate line count and thereby satisfy the context-checkpoint rule of "≥500 lines per append." The evidence rejects this as the wrapping cause: 13 of the 14 context files never reach 500 lines at all (the smallest is 70 lines), so wrapping was plainly not being used to hit the target — if it were, the files would cluster around 500 lines, and they do not. Wrapping and under-filling are therefore two independent defects that merely coexist. The ≥500-line constraint is being systematically violated on its own, which is a genuine bug worth fixing even though it does not explain the wrapping. The fix reinforces this constraint and explicitly forbids using wrapping (or any padding) to game the line count, so that un-wrapping prose cannot tempt an author into re-wrapping to recover lost lines.

## H3 — skill-creator development standard mandates wrapping (REJECTED)

A search of the skill-creator development guidance (README and all reference and agent files) for the terms `wrap`, `line length`, `line-length`, `column`, `80 char`, `one sentence per line`, `hard-wrap`, and `newline` returned zero matches. The development standard does not require or even mention line wrapping. Per the governing rule — if wrapping is not explicitly mandated, it is disallowed — H3 is rejected and wrapping should not occur.

## H4 — an external formatter rewrote the files after writing (REJECTED)

No formatter configuration exists anywhere in the tree: there is no `.editorconfig`, no `.prettierrc` in any form, no `prettier.config`, and no `.markdownlint` file. No configuration file anywhere contains `prose-wrap` or `printWidth`. The project `settings.local.json` and the global `settings.json` contain no `PostToolUse`, `Write`, `Edit`, or formatter hooks. Nothing post-processes the files after Claude Code writes them, so an external tool cannot be responsible for the wrapping.

## Contributing factor (mechanism, not root cause)

Both SKILL.md files embedded example templates whose own prose was hard-wrapped — `context-init` in its Plan Context excerpt and `context-checkpoint` in its Process Summary and Key Findings examples. These wrapped few-shot examples reinforced the H1 habit by demonstrating the unwanted pattern at the exact moment the skill is invoked: the skill simultaneously asked for content and showed that content wrapped. This is not the root cause (H1 operates even without these examples, as the unrelated spec files prove), but it is a concrete amplifier that the fix removes by de-wrapping the example prose inside both skill files.

## Fix summary

The fix adds an explicit "no mid-paragraph line breaks" hard constraint to both skills, de-wraps the embedded example templates so the few-shot demonstration models the correct pattern, and reinforces the ≥500-line constraint with an explicit prohibition on wrap-padding. The fix targets future executions; existing context files are intentionally left unchanged.
