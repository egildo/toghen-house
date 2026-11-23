## House rule: the feature folder is named by date

This overrides the numbering in the step "Create the spec feature directory" below. Ignore `feature_numbering` in `.specify/init-options.json` and ignore any running number.

- The prefix is today's date as `YYYY-MM-DD` (for example `2026-10-25`), then a hyphen, then the short name.
- The folder is `specs/<YYYY-MM-DD>-<short-name>`. Two features on one day differ by their short names.
- If `SPECIFY_FEATURE_DIRECTORY` is given by the user, use it as is.
- Write the resolved folder to `.specify/feature.json` as the step says. Every later command finds the feature there, not by its name.

The spec template in this repository has Findings, Questions and Answers sections. Fill Findings by reading the code, with a path and line for each. Put every decision the owner must make in Questions, with context, alternatives and a recommendation. Leave Answers empty, and the placeholder in its heading as it is: it is the day the owner answers, not the day the spec is written.

## House rule: the spec quality checklist

This replaces the checklist in the step "Specification Quality Validation" below. Write this one to `checklists/requirements.md` instead, and validate the spec against its items. The stock items that forbid implementation details do not hold here: a finding names the code it was read in, by path and line.

```markdown
# Specification Quality Checklist: [FEATURE NAME]

**Purpose**: Validate the spec against the house register before planning
**Created**: [DATE]
**Feature**: [Link to spec.md]

## Findings

- [ ] Every finding ends with its proof, a path and line, and is marked Confirmed, Refuted or Unchecked
- [ ] No Unchecked finding carries a requirement
- [ ] Where a doc and the code disagree, the finding says which one is right

## Requirements

- [ ] Every requirement is testable and rests on a finding or an answered question
- [ ] Success criteria are measurable, and checkable without reading the implementation
- [ ] Edge cases include the W3C structured shapes, not only plain strings
- [ ] Scope is bounded; assumptions and dependencies are named
- [ ] No [NEEDS CLARIFICATION] markers remain

## Questions

- [ ] Every decision the owner must make is a question, with context, two or three alternatives with their costs, and one marked Recommended
- [ ] Answers is empty, and its heading still holds the placeholder for the day the owner answers

## Register

- [ ] Plain, short sentences, one idea each; things are referred to by name
- [ ] No downstream consumer is named
- [ ] All mandatory sections are completed
```

