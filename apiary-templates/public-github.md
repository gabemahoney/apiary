# Template: Public GitHub Project

## New Feature Intake

Public GitHub repos typically receive feature requests and bug reports as GitHub Issues
from external contributors. Private repos usually don't have this external intake channel,
so features are captured through direct conversation with the user.

- github_visibility: whether the project's GitHub repo is public or private

When github_visibility = public:
  - source_references: github resolver
When github_visibility = private or none:
  - source_references: none, interview user

## New Bug Intake

When github_visibility = public:
  - source_references: github resolver
When github_visibility = private or none:
  - source_references: none, interview user

## Documentation Locations

### Reference
Defaults below point at the guidance docs shipped under
`apiary-templates/`. Edit these paths to point at your project's own
docs once you have them; record `omitted` for any category you do
not want to enforce.
- **Engineering best practices**: ../apiary-templates/default-engineering-best-practices.md
- **Test guide**: ../apiary-templates/default-test-guide.md
- **Doc writing guide**: ../apiary-templates/default-doc-writing-guide.md

### Maintained
Ask the user for each (record "omitted" if skipped):
- Contributor docs
- Customer-facing docs
