# Documentation Sync & Review Workflow

Automatically analyze and keep documentation in sync with code changes.

## Overview

This workflow monitors code changes and ensures documentation stays up-to-date. Marty will:
- Detect when code files are modified
- Check if corresponding documentation exists in `web/` folder
- Analyze if documentation accurately reflects current implementation
- Suggest updates or create documentation when missing
- Post review comments on PRs with documentation findings

---

## Use Cases

### 1. **Pull Request Documentation Check**
Automatically review PRs and check if docs are updated:
```yaml
on:
  pull_request:
    types: [opened, synchronize]
```

### 2. **Scheduled Documentation Audit**
Regular checks for documentation drift:
```yaml
on:
  schedule:
    - cron: "0 0 * * 0"  # Weekly
```

### 3. **Manual Documentation Review**
On-demand with workflow_dispatch:
```yaml
on:
  workflow_dispatch:
    inputs:
      target_folder:
        description: "Folder to check (default: web/)"
        required: false
        default: "web/"
```

---

## Workflow File: PR Documentation Check

Save this as `.github/workflows/marty-docs-check.yml`:

```yaml
name: Marty Documentation Check

on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - "src/**"
      - "lib/**"
      - "packages/**"
      - "api/**"
      - "!**/*.md"  # Exclude markdown-only changes
      - "!**/test/**"  # Exclude test files
      - "!**/*.spec.ts"
      - "!**/*.test.ts"

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  docs-check:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history needed for comparison

      - name: Generate Marty token
        id: marty-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.MARTY_APP_ID }}
          private-key: ${{ secrets.MARTY_APP_PRIVATE_KEY }}

      - name: Post progress comment
        id: progress
        run: |
          COMMENT=$(gh pr comment ${{ github.event.pull_request.number }} \
            --body "📚 **Marty is checking documentation...** ⏳" \
            --repo ${{ github.repository }})
          COMMENT_ID=$(echo "$COMMENT" | jq -r '.id')
          echo "comment_id=$COMMENT_ID" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ steps.marty-token.outputs.token }}

      - name: Marty analyzes documentation sync
        uses: nesalia-inc/marty-action@v1.0.0
        with:
          github_token: ${{ steps.marty-token.outputs.token }}
          prompt: |
            REPO: ${{ github.repository }}
            PR NUMBER: ${{ github.event.pull_request.number }}
            PR TITLE: ${{ github.event.pull_request.title }}
            BASE BRANCH: ${{ github.base_ref }}
            HEAD BRANCH: ${{ github.head_ref }}

            You are Marty, checking if documentation stays in sync with code changes.

            ## Your Task

            This PR modifies code files. You need to verify documentation is up-to-date.

            ## Step 1: Identify Changed Files

            Run:
            ```bash
            git diff origin/${{ github.base_ref }}...HEAD --name-only
            ```

            Filter for:
            - Source files: `.ts`, `.js`, `.tsx`, `.jsx`, `.py`, `.go`, etc.
            - Exclude: tests, specs, configs
            - Focus on: API endpoints, functions, components, services

            ## Step 2: Find Corresponding Documentation

            For each changed code file, check if documentation exists in `web/` folder:

            **Mapping examples:**
            - `src/components/Button.tsx` → `web/docs/components/button.md`
            - `api/auth/login.ts` → `web/docs/api/auth/login.md`
            - `lib/utils/format.ts` → `web/docs/utils/format.md`

            Run:
            ```bash
            find web/ -name "*.md" -type f
            ```

            ## Step 3: Analyze Documentation Accuracy

            For each changed code file with documentation:

            1. **Read the code** to understand what it does
            2. **Read the documentation** to see what's documented
            3. **Compare and identify discrepancies:**

            Check for:
            - ✅ **New functions** - Are they documented?
            - ✅ **Changed APIs** - Are params/return types updated?
            - ✅ **Removed features** - Are docs cleaned up?
            - ✅ **Behavior changes** - Are examples still valid?
            - ✅ **Deprecations** - Are they marked in docs?

            ## Step 4: Post Findings

            Edit the progress comment (ID: ${{ steps.progress.outputs.comment_id }}) with:

            ```markdown
            ## 📚 Documentation Review

            ### Changed Files Analyzed: X

            ### ✅ Documentation Status: GOOD / NEEDS UPDATE / MISSING

            #### Findings:

            **Up-to-Date Documentation:**
            - `src/auth.ts` ↔ `web/docs/api/auth.md` ✅

            **Needs Update:**
            - `src/api/users.ts` ↔ `web/docs/api/users.md` ⚠️
              - New `getUserByEmail()` function not documented
              - `deleteUser()` params changed (added `softDelete` param)

            **Missing Documentation:**
            - `src/services/cache.ts` ❌
              - No documentation found in `web/docs/`
              - Suggest creating `web/docs/services/cache.md`

            #### Recommendations:
            - [ ] Update `web/docs/api/users.md` with new function
            - [ ] Create `web/docs/services/cache.md` with API reference
            - [ ] Update examples in `web/docs/guides/caching.md`

            #### Files to Update:
            - `web/docs/api/users.md`
            - `web/docs/services/cache.md` (new)
            ```

            ## Important Guidelines

            - Be specific about what's missing or outdated
            - Provide file paths for documentation updates
            - Suggest documentation structure if missing
            - Use clear formatting (✅ ⚠️ ❌ for status)
            - Focus on user-facing API changes, not implementation details

            ## What Constitutes "Documentation"

            **Worth documenting:**
            - Public API endpoints
            - Component props and usage
            - Service interfaces
            - Configuration options
            - Breaking changes

            **Skip documentation:**
            - Internal utility functions
            - Test files
            - Implementation details
            - Private methods

          claude_args: |
            --allowedTools "Bash(gh:*),Bash(git:*),Bash(find:*),Bash(grep:*),Bash(jq:*),Read,Edit,Write"
            --max-turns 25

        env:
          # Z.ai Configuration
          ANTHROPIC_BASE_URL: https://api.z.ai/api/anthropic
          ANTHROPIC_AUTH_TOKEN: ${{ secrets.ZAI_API_KEY }}
          ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.7
          ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
          ANTHROPIC_DEFAULT_OPUS_MODEL: glm-4.7
```

---

## Workflow File: Auto-Update Documentation

Save this as `.github/workflows/marty-docs-sync.yml`:

```yaml
name: Marty Documentation Sync

on:
  pull_request:
    types: [opened, synchronize]
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write
  id-token: write

jobs:
  sync-docs:
    if: contains(github.event.pull_request.body, '[marty: sync docs]') || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.ref }}
          fetch-depth: 0

      - name: Generate Marty token
        id: marty-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.MARTY_APP_ID }}
          private-key: ${{ secrets.MARTY_APP_PRIVATE_KEY }}

      - name: Configure git
        run: |
          git config --global user.name "marty[bot]"
          git config --global user.email "2883066+marty[bot]@users.noreply.github.com"

      - name: Post progress
        id: progress
        run: |
          COMMENT=$(gh pr comment ${{ github.event.pull_request.number }} \
            --body "📚 **Marty is syncing documentation...** ⏳")
          echo "comment_id=$(echo $COMMENT | jq -r '.id')" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ steps.marty-token.outputs.token }}

      - name: Marty syncs documentation
        uses: nesalia-inc/marty-action@v1.0.0
        timeout-minutes: 30
        with:
          github_token: ${{ steps.marty-token.outputs.token }}
          prompt: |
            REPO: ${{ github.repository }}
            PR NUMBER: ${{ github.event.pull_request.number }}

            You are Marty, automatically updating documentation to match code changes.

            ## Your Task

            1. Find all changed code files (exclude tests)
            2. For each file, check if documentation exists in `web/`
            3. Update existing documentation or create new files
            4. Commit changes to the PR branch

            ## Process

            ### Step 1: Identify Changed Code
            ```bash
            git diff origin/main...HEAD --name-only | grep -E '\.(ts|js|tsx|jsx|py)$'
            ```

            ### Step 2: Map Code to Documentation

            **Rules:**
            - `src/api/*` → `web/docs/api/*.md`
            - `src/components/*` → `web/docs/components/*.md`
            - `lib/services/*` → `web/docs/services/*.md`
            - Create folders if they don't exist

            ### Step 3: Analyze Code

            For each changed file:
            - Extract public functions/classes/components
            - Identify parameters, return types, props
            - Note usage examples

            ### Step 4: Update/Create Documentation

            **If docs exist:**
            - Update API references
            - Fix outdated examples
            - Add new functions/features
            - Remove deprecated items

            **If docs don't exist:**
            - Create new markdown file with:
              ```markdown
              # Filename

              ## Overview
              Brief description

              ## API Reference
              ### FunctionName
              **Parameters:**
              - `param1`: description
              - `returns`: description

              ## Example
              ```typescript
              // usage example
              ```

              ## Notes
              Additional context
              ```

            ### Step 5: Commit Changes

            ```bash
            git add web/docs/
            git commit -m "docs: Update documentation for code changes"
            git push
            ```

            ## Documentation Style

            - Use clear, concise language
            - Include code examples
            - Document parameters/props/return values
            - Note breaking changes
            - Link related docs

            ## What to Document

            ✅ **Document:**
            - Public API endpoints
            - Component props and usage
            - Service interfaces
            - Configuration options

            ❌ **Skip:**
            - Internal utilities
            - Private methods
            - Test helpers

          claude_args: |
            --allowedTools "Bash(gh:*),Bash(git:*),Bash(find:*),Read,Write,Edit"
            --max-turns 40

        env:
          ANTHROPIC_BASE_URL: https://api.z.ai/api/anthropic
          ANTHROPIC_AUTH_TOKEN: ${{ secrets.ZAI_API_KEY }}
          ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.7

      - name: Update progress comment
        run: |
          gh pr comment ${{ github.event.pull_request.number }} \
            --edit ${{ steps.progress.outputs.comment_id }} \
            --body "✅ **Documentation synced successfully!**

            Updated/created documentation files in `web/docs/` to match code changes.

            Check the PR for the documentation updates."
        env:
          GH_TOKEN: ${{ steps.marty-token.outputs.token }}
```

---

## Usage Examples

### Example 1: PR with Missing Documentation

**User creates PR:**
> Changes login API to add 2FA support

**Marty posts:**
> ## 📚 Documentation Review
>
> ### Changed Files Analyzed: 3
>
> ### ✅ Documentation Status: NEEDS UPDATE
>
> #### Findings:
>
> **Missing Documentation:**
> - `src/api/auth/login.ts` ❌
>   - New `enable2FA()` function not documented
>   - `login()` params changed (added `totpCode`)
>   - New error codes not documented
>
> #### Recommendations:
> - [ ] Add `enable2FA()` to `web/docs/api/auth.md`
> - [ ] Update `login()` params in documentation
> - [ ] Document new error codes (`INVALID_TOTP`, `TOTP_RATE_LIMITED`)
>
> #### Files to Update:
> - `web/docs/api/auth.md`

---

### Example 2: Auto-Sync Documentation

**User comments on PR:**
> @martyy-code [marty: sync docs]

**Marty responds:**
> 📚 **Marty is syncing documentation...** ⏳
>
> *[Analyzes code, updates/creates docs]*
>
> ✅ **Documentation synced successfully!**
>
> Updated:
> - `web/docs/api/auth.md` - Added 2FA functions
> - `web/docs/guides/authentication.md` - Added 2FA setup guide
> - `web/docs/api/errors.md` - Added new error codes

---

### Example 3: Scheduled Documentation Audit

**Weekly workflow runs and creates an issue:**

> ## 📚 Weekly Documentation Audit
>
> **Date:** 2024-02-17
>
> ### Documentation Status: NEEDS ATTENTION
>
> #### Issues Found:
>
> **Outdated Documentation:**
> - `web/docs/api/users.md` - Missing new `getUserByEmail()` function
> - `web/docs/components/Button.md` - Props don't match current implementation
>
> **Missing Documentation:**
> - `src/services/analytics.ts` - No docs found
> - `src/hooks/useWebSocket.ts` - No docs found
>
> #### Action Items:
> - [ ] Update user API docs
> - [ ] Update Button component docs
> - [ ] Create analytics service docs
> - [ ] Create WebSocket hook docs
>
> **Files to Update:** 4
> **Estimated Time:** 2 hours

---

## Configuration Options

### 1. **Custom Documentation Folder**

Change `web/` to your docs location:

```yaml
prompt: |
  Check documentation in `docs/` folder (instead of web/)
```

### 2. **File Mapping Rules**

Customize how code maps to documentation:

```yaml
prompt: |
  **Custom mapping rules:**
  - `apps/api/*` → `docs/reference/api/*.md`
  - `packages/ui/*` → `docs/components/*.md`
  - `shared/utils/*` → `docs/utils/*.md`
```

### 3. **Severity Levels**

Configure how strict Marty should be:

```yaml
prompt: |
  ## Severity Levels

  **Critical** - Block PR merge if:
  - Breaking changes without docs
  - New public APIs undocumented

  **Warning** - Note in PR if:
  - Minor API changes undocumented
  - Examples outdated

  **Info** - Log if:
  - Internal code changed (no docs needed)
```

---

## Advanced Features

### 1. **Multi-Language Documentation**

```yaml
prompt: |
  Check documentation in multiple languages:
  - `web/docs/en/` (English)
  - `web/docs/fr/` (French)
  - `web/docs/es/` (Spanish)

  Ensure all languages are updated when code changes.
```

### 2. **Version-Specific Documentation**

```yaml
prompt: |
  For versioned documentation:
  - Code in `v2/` → docs in `web/docs/v2/`
  - Code in `v3/` → docs in `web/docs/v3/`
  - Check for migration guides between versions
```

### 3. **Interactive Documentation**

```yaml
prompt: |
  Check if interactive examples work:
  - Find code examples in documentation
  - Extract them to temporary files
  - Run tests/linting on examples
  - Flag examples that don't compile
```

---

## Integration with Other Workflows

### With PR Review Workflow

```yaml
# In PR Review workflow, add:
prompt: |
  In addition to code review, check:
  - Are API changes documented?
  - Are examples up-to-date?
  - Should documentation be updated?

  Add a "Documentation Review" section to your review comment.
```

### With Issue Triage Workflow

```yaml
# Add labels to issues about documentation
if: contains(github.event.issue.body, 'documentation')
  then add label 'docs'
```

---

## Best Practices

### 1. **Documentation First**

Encourage developers to write docs with code:

```yaml
prompt: |
  If PR has code changes but no doc changes:
  - Politely remind author to update docs
  - Provide template for missing documentation
  - Don't block PR, just flag it
```

### 2. **Incremental Updates**

```yaml
prompt: |
  Update documentation incrementally:
  - Only update sections related to changed code
  - Don't rewrite entire docs
  - Preserve existing structure and style
```

### 3. **Consistent Formatting**

```yaml
prompt: |
  Follow repository documentation style:
  - Use existing markdown formatting
  - Match heading levels
  - Preserve code block style
  - Keep consistent tone
```

---

## Troubleshooting

### Issue: False positives - Marty says docs are outdated when they're not

**Cause:** Marty doesn't understand your documentation structure

**Solution:** Add clear mapping rules and examples in prompt

---

### Issue: Documentation created in wrong location

**Cause:** File mapping logic doesn't match your folder structure

**Solution:** Customize the mapping rules in the prompt

---

### Issue: Too many files flagged

**Cause:** Checking all files including internals

**Solution:** Filter to only public-facing code:
```yaml
prompt: |
  Only check documentation for:
  - Files in `src/api/` (public endpoints)
  - Files in `src/components/` (UI components)
  - Files with `@public` JSDoc tags

  Skip:
  - Files in `src/internal/`
  - Files with `@private` tags
  - Test files
```

---

## Performance Tips

### 1. **Limit Analysis Scope**

```yaml
on:
  pull_request:
    paths:
      - "src/api/**"  # Only check API changes
      - "src/components/**"  # Only check components
```

### 2. **Use Faster Model**

```yaml
env:
  ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
```

### 3. **Reduce Max Turns**

```yaml
claude_args: |
  --max-turns 15  # For simple doc checks
```

---

## Metrics to Track

| Metric | How to Measure |
|--------|----------------|
| Docs up-to-date % | (PRs with updated docs / Total PRs) × 100 |
| Missing docs per PR | Average count of undocumented changes |
| Auto-sync accuracy | (Correct updates / Total auto-updates) × 100 |
| Review time saved | Hours saved by auto-documentation |

---

## Related Workflows

- **[PR Review](./PR_REVIEW_WORKFLOW.md)** - Include doc review in code review
- **[Issue Triage](./ISSUE_TRIAGE_WORKFLOW.md)** - Tag documentation issues
- **[Implementation](./IMPLEMENTATION_WORKFLOW.md)** - Auto-generate docs for new code

---

## Quick Start

1. **Add the workflow** to `.github/workflows/marty-docs-check.yml`
2. **Customize the paths** to match your repository structure
3. **Test on a PR** with code changes
4. **Review Marty's findings** and adjust prompts as needed
5. **Optionally add auto-sync** workflow for automatic updates

---

## Support

For issues or questions:
- Check main [Marty documentation](./README.md)
- Review [troubleshooting guide](./docs/faq.md)
- Open an issue on [Marty repository](https://github.com/nesalia-inc/marty-action)
