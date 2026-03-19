# How to Request Spec Changes

Place your new spec file in the project, then type into Claude Code:

```
/change-spec [path to new spec file]
```

## Examples

```
/change-spec staff-search-docs/spec/main_app_specification_ja.md
```

```
/change-spec ~/Downloads/new-spec-v2.pdf
```

## What happens

1. Claude reads your new spec and all current ticket specs
2. Claude generates a **Change Request Report** with 3 sections:

   **1) Issues** — differences between current spec and new spec (new / changed / removed features)

   **2) Impact Scope** — which layers are affected per issue (Frontend, Backend, Database, UI/UX, other tickets)

   **3) Tasks + Estimated Effort** — concrete tasks, files to change, and estimated dev-days for each issue

3. You review the report and choose which changes to apply
4. Claude updates the ticket specs and index

## Output

The report is saved as `staff-search-docs/screens/_CHANGE_REQUEST_[DATE].md` for tracking.
