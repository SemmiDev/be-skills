---
name: git-commit-formatter
description: Formats git commit messages according to Conventional Commits specification using proper English language.
---

# Git Commit Formatter Skill

When writing a git commit message, you MUST follow the Conventional Commits specification.
You MUST output the description in **English** that is grammatically correct and concise.

## Format

`<type>[optional scope]: <description>`

## Allowed Types (Do not translate the Type tags)

- **feat**: A new feature
- **fix**: A bug fix
- **docs**: Documentation only changes
- **style**: Formatting changes that do not affect logic (whitespace, semicolons, etc.)
- **refactor**: Code changes that neither fix bugs nor add features
- **perf**: Code changes that improve performance
- **test**: Adding or fixing tests
- **chore**: Changes to build process or auxiliary tools

## Instructions

1. **Type & Scope:**
   - Analyze the changes to determine the primary `type`.
   - **KEEP the `type` in English** (e.g., `feat`, not translated).
   - Identify the `scope` if applicable (e.g., component name, filename).

2. **Description (English):**
   - Write the description strictly in **English**.
   - Use **clear, active voice**.
   - Start with a **base verb** (e.g., "add", "fix", "remove", "update").
   - Avoid slang or ambiguous abbreviations.
   - Keep it concise (under 72 characters if possible).

3. **Breaking Changes:**
   - If there are breaking changes, add a footer starting with  
     `BREAKING CHANGE:` followed by the description in English.

## Examples

**Good:**
`feat(auth): add Google login support`  
`fix(api): handle null payload validation`  
`style(css): fix indentation in global styles`  
`docs: update installation guide in README`

**Bad (Avoid these):**
`feat: adding login feature` (not concise / not imperative)  
`fix: fixed bug` (redundant wording)  
`fitur(auth): login google` (translated type tag - do not do this)
