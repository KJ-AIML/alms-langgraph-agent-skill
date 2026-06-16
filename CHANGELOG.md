# Changelog

## 0.3.0 - 2026-06-16

- Added `Do Not Overbuild` section with Simple Agent vs Production Workflow decision table.
- Added `Factory Compatibility` section documenting `create_*_agent()` coexistence with `AgentManager`.
- Added `Future Domain-Module Compatibility` section for layer-first to domain-module migration path.
- Added `Final Response Format` section for AI coding agent output.
- Added `Do Not Overbuild` decision table and factory compat note to `references/alms-patterns.md`.
- Bumped skill version to `0.3.0`. Compatible with `alms >=0.2.1`.

## 0.2.1 - 2026-05-12

- Shortened the skill frontmatter description for more precise activation.
- Added `version` and `compatible_with` metadata.
- Documented compatibility with `alms >=0.2.1`.
- Clarified that missing prompt manager, markdown prompt, schema, or workflow skeleton directories should be created before implementing production LangGraph behavior.
