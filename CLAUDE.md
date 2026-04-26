# Skills Repository

## Plugin Version Bumping

**Every PR that touches files under `plugins/<name>/` must bump the version in `plugins/<name>/.claude-plugin/plugin.json`.**

Version bump rules (semver):

- **Bug fix** (corrects incorrect behavior, fixes a defect in an existing skill step): bump the **patch** version (e.g., `2.5.0` → `2.5.1`).
- **New feature** (adds new behavior, new steps, new routing, new fallback logic): bump the **minor** version (e.g., `2.5.0` → `2.6.0`).
- **Breaking change** (removes or renames a skill, changes the skill interface in a backwards-incompatible way): bump the **major** version (e.g., `2.5.0` → `3.0.0`).

Individual skill files also carry their own `version:` field in their YAML front matter. Bump the skill version to match the same increment applied to the plugin when a PR modifies that skill.

PRs that only modify `README.md`, `CLAUDE.md`, or non-plugin files do not require a plugin version bump.
