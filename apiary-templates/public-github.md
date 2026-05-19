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
Ask the user for each (record "omitted" if skipped). For each category,
offer three options:
- Point at an existing path in the project.
- Use the bundled default (project-setup will copy
  ~/.claude/apiary-templates/default-<category>.md into the project under
  docs/apiary-defaults/<category>.md and record the resulting
  project-relative path).
- omitted.

Categories:
- Engineering best practices
- Test guide
- Doc writing guide

### Maintained
Ask the user for each (record "omitted" if skipped):
- Contributor docs
- Customer-facing docs
