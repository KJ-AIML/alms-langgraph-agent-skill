# Changelog

## 0.4.0 — 2026-06-28

- Skill is now profile-aware: reads `[tool.alms]` from `pyproject.toml` before generating code.
- Added "Profile / Capability Contract" section in SKILL.md with capability rules.
- Updated "Core Workflow" to start with profile inspection and capability-aware directory scanning.
- Updated "Do Not Overbuild" to include capability-based constraints.
- Made "Project Shape" section in references/alms-patterns.md conditional on active capabilities.
- Added capability gates to all production workflow recipes in references/alms-patterns.md.
- Updated "Run And Verify" with profile-specific verification commands.
- Added "Profile Impact" field to Final Response Format.
- Bumped version to `0.4.0`. Compatible with `alms >=0.3.0`.
- Backward compatible: `full` profile preserves all v0.3.0 behaviour.

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
