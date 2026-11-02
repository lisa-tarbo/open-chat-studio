# Developer Docs Review Checklist

Use this checklist when writing or reviewing a page under `docs/`. Each item has a reason so a reviewer can judge edge cases. Skip an item only when it does not apply to the page.

For prose style, follow the [Writing style](https://github.com/dimagi/open-chat-studio/blob/main/AGENTS.md#writing-style) section of `AGENTS.md`.

## Audience and scope

- [ ] **The opening paragraph says what the page covers and who it is for.** A reader should know within two sentences whether they are in the right place.
- [ ] **Content owned elsewhere is linked, not restated.** This covers workflow header comments, READMEs, `AGENTS.md`, other guides and user docs. Restated content drifts.
- [ ] **Features with user-facing docs link to them.** Say which page covers the user's view and which covers the implementation.

## Accuracy

- [ ] **Every command, path, setting, flag and symbol was checked against source.** The PR description says what was checked and how.
- [ ] **The page avoids values that drift.** Counts, versions and line numbers go stale. When one is needed, name its source of truth.
- [ ] **The page describes current behaviour only.** No "this PR", "now", "no longer" or "previously". Limitations and planned work go in a labelled section with an issue link.

## Content

- [ ] **Non-obvious rules give their reason in one sentence, next to the rule.** A reader who knows why can handle cases the page did not anticipate.
- [ ] **Procedures are numbered, one action per step.** Where a failure is silent, the step says how to verify the result.
- [ ] **Warnings come before the step they apply to.** A warning after the step is read too late.
- [ ] **Setup and how-to guides include troubleshooting for known failures.** List symptoms and the fix.
- [ ] **Examples are minimal and match the current codebase.** Show an incorrect example only for a mistake readers are likely to make.

## Voice and tone

- [ ] **Steps are imperative and the reader is "you".** Describe system behaviour with "OCS" or the named component, not "we" ("OCS verifies the signature"). "We" is ambiguous for external contributors and self-hosters.
- [ ] **UI labels are bold and match the UI exactly. Code identifiers are in backticks.** If the page uses both the product name "Chatbot" and the code name `Experiment`, say so once.

## Formatting

- [ ] **The H1 is the topic in Title Case. H2 and lower are sentence case.** Name the topic, not the document type: "Versioning", not "Versioning Dev Documentation".
- [ ] **Every code fence has a language.** Use `bash` (not `shell`), `python`, `text` and so on. Mark correct and incorrect examples in words, not ✅/❌ in code comments.
- [ ] **Callouts use `!!! note` or `!!! warning`.** Not bold prefixes such as **Note:** or **Important:**.
- [ ] **Lookups are tables.** Parameters, flags, grants and states are easier to scan in a table than in prose.
- [ ] **Link text describes the target.** Never "here" or a bare URL.
