# End-to-End AI Development Workflow with Marty

This document describes a complete autonomous development workflow where Marty analyzes issues, discusses solutions, implements features, creates pull requests, and reviews code - all automatically.

## 🎯 Overview

The workflow enables a seamless development experience where developers can:

1. **Create an issue** → Marty automatically triages and clarifies requirements
2. **Discuss with Marty** → Interactive dialogue to refine specifications
3. **Request implementation** → Marty codes, creates PR, and self-reviews
4. **Review and merge** → Human approves the final result

---

## 📋 The Complete Workflow

```
User opens issue
    ↓
Marty auto-analyzes & categorizes
    ↓
Interactive dialogue (User ↔ Marty)
    ↓
User: "@marty implements this"
    ↓
Marty creates branch, codes, commits
    ↓
Marty creates PR
    ↓
Marty auto-reviews the PR
    ↓
Human reviews & merges
```

---

## 🚀 Architecture

You need **4 separate GitHub Actions workflows**:

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| **Issue Triage** | `issues: [opened, edited]` | Auto-analyze new issues |
| **Issue Response** | `issue_comment: [created]` + `@marty` | Answer questions and clarify |
| **Implementation** | `issue_comment: [created]` + `@marty implements` | Code and create PR |
| **PR Review** | `pull_request: [opened, synchronize]` | Auto-review PRs |

---

## 1️⃣ Issue Triage Workflow

**File:** `.github/workflows/marty-issue-triage.yml`

### Purpose

Automatically analyze newly created issues to:
- Categorize (bug/feature/question/refactoring)
- Ask clarifying questions if requirements are unclear
- Estimate complexity
- Add appropriate labels
- Suggest implementation approach

### YAML Configuration

```yaml
name: Marty Issue Triage

on:
  issues:
    types: [opened, edited]

permissions:
  contents: read
  issues: write
  id-token: write

jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - name: Generate Marty token
        id: marty-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.MARTY_APP_ID }}
          private-key: ${{ secrets.MARTY_APP_PRIVATE_KEY }}

      - name: Marty Issue Analysis
        uses: nesalia-inc/marty-action@v1.0.0
        with:
          github_token: ${{ steps.marty-token.outputs.token }}
          prompt: |
            REPO: ${{ github.repository }}
            ISSUE NUMBER: ${{ github.event.issue.number }}
            ISSUE TITLE: ${{ github.event.issue.title }}
            ISSUE BODY: ${{ github.event.issue.body }}
            AUTHOR: ${{ github.event.issue.user.login }}

            You are Marty, an intelligent development assistant. Analyze this issue:

            ## Analysis Tasks

            1. **Categorize**: Is this a bug, feature request, question, or refactoring?
            2. **Clarity Check**: Is the request clear and specific enough?
            3. **Missing Info**: What additional information is needed?
            4. **Complexity**: Estimate (trivial/simple/complex/very-complex)
            5. **Feasibility**: Can this be implemented automatically?

            ## Labels to Add

            Add appropriate labels using:
            `gh issue edit $NUMBER --add-label "label1,label2,label3"`

            Suggested labels:
            - Type: `bug`, `feature`, `question`, `refactoring`, `documentation`
            - Complexity: `complexity-trivial`, `complexity-simple`, `complexity-complex`
            - Priority: `priority-critical`, `priority-high`, `priority-medium`, `priority-low`
            - Component: Add component-specific labels if applicable

            ## Response Requirements

            If the issue is UNCLEAR:
            - Ask specific questions in a comment
            - Don't add labels yet (wait for clarification)

            If the issue is CLEAR:
            - Add appropriate labels
            - Post a comment confirming understanding
            - If complex, suggest breaking it down into sub-tasks
            - Estimate implementation time

            Keep responses concise and friendly.

          claude_args: |
            --allowedTools "Bash(gh issue:*),Bash(gh label:*),Bash(gh api:*)"
            --max-turns 10

        env:
          # Z.ai Configuration
          ANTHROPIC_BASE_URL: https://api.z.ai/api/anthropic
          ANTHROPIC_AUTH_TOKEN: ${{ secrets.ZAI_API_KEY }}
          ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.7
          ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
          ANTHROPIC_DEFAULT_OPUS_MODEL: glm-4.7
```

### Example Output

**User opens issue:**
> Add user authentication with JWT

**Marty responds:**
> Hi! I'd like to clarify a few things before implementing:
> 1. JWT expiration time?
> 2. Refresh token rotation needed?
> 3. Two-factor authentication support?
> 4. Which endpoints need protection?
>
> Tagged as `feature`, `complexity-complex`, `auth`

---

## 2️⃣ Issue Response Workflow

**File:** `.github/workflows/marty-issue-response.yml`

### Purpose

Enable interactive dialogue on issues. Whenever someone mentions `@marty`, Marty responds intelligently with full context of the issue history.

### YAML Configuration

```yaml
name: Marty Issue Response

on:
  issue_comment:
    types: [created]

permissions:
  contents: read
  issues: write
  id-token: write

jobs:
  respond:
    if: contains(github.event.comment.body, '@marty')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate Marty token
        id: marty-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.MARTY_APP_ID }}
          private-key: ${{ secrets.MARTY_APP_PRIVATE_KEY }}

      - name: Marty Responds to Comment
        uses: nesalia-inc/marty-action@v1.0.0
        with:
          github_token: ${{ steps.marty-token.outputs.token }}
          prompt: |
            REPO: ${{ github.repository }}
            ISSUE NUMBER: ${{ github.event.issue.number }}
            COMMENT AUTHOR: ${{ github.event.comment.user.login }}
            COMMENT: ${{ github.event.comment.body }}

            You are Marty, mentioned in an issue comment.

            ## Your Task

            1. Read the ENTIRE issue history (title, body, all previous comments)
            2. Understand the full context and current state of discussion
            3. Respond intelligently to the latest comment
            4. If someone asks questions, answer them clearly
            5. If technical discussion, provide helpful insights
            6. If implementation is discussed, offer to help implement when ready

            ## Important Guidelines

            - Be concise but thorough
            - Reference specific code or files when helpful
            - Ask clarifying questions if something is unclear
            - Be friendly and professional
            - Don't mention that you're an AI (just be helpful)

            Post your response as a comment on the issue.

          claude_args: |
            --allowedTools "Bash(gh issue:*),Bash(gh api:*),Read"
            --max-turns 8

        env:
          # Z.ai Configuration
          ANTHROPIC_BASE_URL: https://api.z.ai/api/anthropic
          ANTHROPIC_AUTH_TOKEN: ${{ secrets.ZAI_API_KEY }}
          ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.7
          ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
          ANTHROPIC_DEFAULT_OPUS_MODEL: glm-4.7
```

### Example Dialogue

**User:**
> @marty 24h expiration, no refresh tokens, simple implementation

**Marty:**
> Got it! Simple JWT authentication with:
> - 24-hour token expiration
> - No refresh token rotation
> - Basic login and verify endpoints
>
> I'll create:
> - `/api/auth/login` - POST endpoint for login
> - `/api/auth/verify` - GET endpoint to verify tokens
> - JWT middleware for route protection
>
> Ready to implement when you are! Just say "@marty implements this"

---

## 3️⃣ Implementation Workflow

**File:** `.github/workflows/marty-implementation.yml`

### Purpose

When requested with `@marty implements` (or similar), Marty:
1. Creates a new branch (`marty/issue-XXX`)
2. Writes the code
3. Commits changes
4. Pushes the branch
5. Creates a pull request
6. Posts confirmation on the issue

### YAML Configuration

```yaml
name: Marty Implementation

on:
  issue_comment:
    types: [created]

permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write

jobs:
  implement:
    # Trigger only for implementation requests
    if: |
      contains(github.event.comment.body, '@marty') &&
      (contains(github.event.comment.body, 'implements') ||
       contains(github.event.comment.body, 'implémentes') ||
       contains(github.event.comment.body, 'create') ||
       contains(github.event.comment.body, 'créer') ||
       contains(github.event.comment.body, 'code') ||
       contains(github.event.comment.body, 'coder'))

    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history needed for branching
          token: ${{ secrets.MARTY_APP_PRIVATE_KEY }}  # Use Marty's token for pushing

      - name: Generate Marty token
        id: marty-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.MARTY_APP_ID }}
          private-key: ${{ secrets.MARTY_APP_PRIVATE_KEY }}

      - name: Configure Git as Marty
        run: |
          git config --global user.name "marty[bot]"
          git config --global user.email "2883066+marty[bot]@users.noreply.github.com"

      - name: Marty Implements the Issue
        uses: nesalia-inc/marty-action@v1.0.0
        timeout-minutes: 45  # Long timeout for complex implementations
        with:
          github_token: ${{ steps.marty-token.outputs.token }}
          prompt: |
            REPO: ${{ github.repository }}
            ISSUE NUMBER: ${{ github.event.issue.number }}
            ISSUE TITLE: ${{ github.event.issue.title }}
            ISSUE BODY: ${{ github.event.issue.body }}
            COMMENT AUTHOR: ${{ github.event.comment.user.login }}
            LATEST COMMENT: ${{ github.event.comment.body }}

            You are Marty, an AI development assistant. You've been asked to implement this issue.

            ## Implementation Process

            1. **Understand Requirements**
               - Read the entire issue history
               - Understand what needs to be built
               - Identify files that need changes

            2. **Explore Codebase**
               - Read relevant files to understand architecture
               - Follow existing patterns and conventions
               - Identify dependencies and integrations

            3. **Create Branch**
               ```bash
               git checkout -b marty/issue-${{ github.event.issue.number }}
               ```

            4. **Implement Solution**
               - Write clean, well-documented code
               - Follow project coding standards
               - Add error handling
               - Include comments for complex logic

            5. **Add Tests** (if applicable)
               - Unit tests for new functions
               - Integration tests for new endpoints
               - Ensure existing tests still pass

            6. **Commit Changes**
               ```bash
               git add .
               git commit -m "feat: Resolve #${{ github.event.issue.number }} - <brief description>

               - <detailed changelog>
               - Closes #${{ github.event.issue.number }}"
               ```

            7. **Push Branch**
               ```bash
               git push origin marty/issue-${{ github.event.issue.number }}
               ```

            8. **Create Pull Request**
               ```bash
               gh pr create \
                 --title "[Marty] Resolve #${{ github.event.issue.number }}: <title>" \
                 --body "## Summary

               Implements #${{ github.event.issue.number }}.

               ## Changes

               - <change 1>
               - <change 2>
               - <change 3>

               ## Testing

               - <testing approach>

               Closes #${{ github.event.issue.number }}" \
                 --base main
               ```

            9. **Confirm on Issue**
               - Post a comment on the original issue
               - Include link to the created PR
               - Summary of what was implemented

            ## Important Guidelines

            - **Code Quality**: Write production-ready code, not prototypes
            - **Testing**: Add tests for non-trivial changes
            - **Documentation**: Update README/API docs if needed
            - **Git Hygiene**: Clear commit messages, proper branch naming
            - **Error Handling**: Handle edge cases and errors gracefully
            - **Security**: No hardcoded secrets, proper input validation
            - **Performance**: Consider performance implications

            ## If You Get Stuck

            - If requirements are unclear, ask on the issue
            - If technical blockers exist, explain them
            - If issue is too complex, suggest breaking it down
            - Always communicate what you're doing

            Start by reading the issue and understanding what needs to be done.

          claude_args: |
            --allowedTools "Read,Write,Edit,Bash(git:*),Bash(gh pr:*),Bash(gh issue:*),Bash(gh api:*)"
            --max-turns 60

        env:
          # Z.ai Configuration
          ANTHROPIC_BASE_URL: https://api.z.ai/api/anthropic
          ANTHROPIC_AUTH_TOKEN: ${{ secrets.ZAI_API_KEY }}
          ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.7
          ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
          ANTHROPIC_DEFAULT_OPUS_MODEL: glm-4.7

      - name: Implementation Complete
        if: success()
        run: |
          echo "✅ Implementation completed successfully!"
          echo "Check the PR for Marty's self-review"

      - name: Implementation Failed
        if: failure()
        run: |
          echo "❌ Implementation failed"
          echo "Check the workflow logs for details"
```

### Example Output

**User:**
> @marty implements this issue

**Marty (in workflow logs):**
```
✓ Created branch marty/issue-123
✓ Read existing auth code structure
✓ Implemented /api/auth/login endpoint
✓ Implemented /api/auth/verify endpoint
✓ Created JWT middleware
✓ Added input validation
✓ Added error handling
✓ Written unit tests
✓ All tests passing
✓ Committed changes
✓ Pushed branch
✓ Created PR #45
✓ Posted confirmation on issue
```

**Marty posts on issue:**
> ✅ Implementation complete! I've created PR #45.
>
> **What I built:**
> - JWT authentication with 24h expiration
> - `/api/auth/login` endpoint
> - `/api/auth/verify` endpoint
> - JWT middleware for route protection
> - Input validation and error handling
> - Unit tests with 95% coverage
>
> **View PR:** [link]
>
> I've also performed an initial code review on the PR. Please review and merge if it looks good!

---

## 4️⃣ PR Review Workflow

**File:** `.github/workflows/marty-pr-review.yml` (Already documented in [PR_REVIEW_WORKFLOW.md](./PR_REVIEW_WORKFLOW.md))

### Purpose

Automatically review every pull request (including those created by Marty):
- Code quality analysis
- Security checks
- Performance considerations
- Best practices verification
- Inline comments for specific issues
- Summary comment with findings

### Integration with Implementation

When Marty creates a PR, the PR Review workflow **automatically triggers** and reviews Marty's own code. This creates a self-checking mechanism where Marty reviews its own work before human review.

---

## 🎨 Complete Example Walkthrough

### Step 1: User Opens Issue

```
Issue #123: Add user authentication with JWT tokens

Description:
We need JWT-based authentication for our API.
Should support login and token verification.
```

**Workflow:** `marty-issue-triage.yml` triggers

**Marty responds:**
> Hi! I can help implement JWT authentication. A few questions:
>
> 1. **Token expiration**: How long should tokens be valid? (1h, 24h, 7 days?)
> 2. **Refresh tokens**: Do we need refresh token rotation?
> 3. **Protected routes**: Which endpoints need authentication?
> 4. **User storage**: Where are user credentials stored?
>
> Once I have these details, I can implement this for you!
>
> Labels added: `feature`, `auth`, `needs-clarification`

---

### Step 2: User Provides Details

```
Comment: 24h token expiration, no refresh tokens needed.
All /api/* endpoints except /api/auth/* need protection.
Users are in PostgreSQL database.
```

**Workflow:** `marty-issue-response.yml` triggers

**Marty responds:**
> Perfect! Here's what I'll implement:
>
> **Endpoints:**
> - `POST /api/auth/login` - Email/password authentication
> - `GET /api/auth/verify` - Verify token validity
>
> **Middleware:**
> - JWT authentication middleware for protected routes
>
> **Database:**
> - Query users from PostgreSQL
> - Verify password with bcrypt
>
> **Security:**
> - Environment variable for JWT secret
> - Password hashing with bcrypt
> - Input validation on all endpoints
>
> Ready when you are! Just say "@marty implements this"
>
> Labels updated: `feature`, `auth`, `ready-to-implement`, `complexity-medium`

---

### Step 3: User Requests Implementation

```
Comment: @marty implements this issue
```

**Workflow:** `marty-implementation.yml` triggers

**Marty (process):**
```
1. Read issue #123 and all comments
2. Explore codebase structure
3. Read existing database connection code
4. Create branch: marty/issue-123
5. Implement JWT utilities (jwt.js)
6. Implement auth controller (auth.controller.js)
7. Implement auth middleware (auth.middleware.js)
8. Add routes (auth.routes.js)
9. Update main server.js to include auth routes
10. Write unit tests for each module
11. Test all endpoints
12. Commit: "feat: Add JWT authentication"
13. Push branch
14. Create PR #45 with full description
15. Comment on issue #123 with PR link
```

**Marty posts on issue:**
> ✅ **Implementation Complete!**
>
> I've created PR #45 with the JWT authentication system.
>
> **What's included:**
> - ✅ JWT login endpoint (`/api/auth/login`)
> - ✅ Token verification endpoint (`/api/auth/verify`)
> - ✅ Authentication middleware for protected routes
> - ✅ PostgreSQL user queries
> - ✅ Bcrypt password hashing
> - ✅ Input validation
> - ✅ Error handling
> - ✅ Unit tests (95% coverage)
>
> **Branch:** `marty/issue-123`
> **PR:** [link to PR #45]
>
> I've performed an initial code review on the PR. Please review and merge if everything looks good!

---

### Step 4: Marty Self-Reviews the PR

**Workflow:** `marty-pr-review.yml` automatically triggers on PR creation

**Marty posts on PR #45:**
> ## Code Review Summary
>
> ### ✅ Overall Assessment
> The implementation is solid and follows best practices. Code is clean, well-documented, and properly tested.
>
> ### 🔴 Critical Issues
> **None found**
>
> ### 🟡 Recommendations
> 1. **Line 45** - Consider adding rate limiting on `/api/auth/login` to prevent brute force attacks
> 2. **Line 78** - JWT secret validation on startup would fail fast if misconfigured
> 3. **Performance** - Consider caching user queries in Redis for high-traffic scenarios
>
> ### ✅ Positive Notes
> - Excellent error handling throughout
> - Comprehensive input validation
> - Good separation of concerns (controller/middleware/routes)
> - High test coverage (95%)
> - Clear documentation and comments
> - Proper environment variable usage
> - Security best practices followed (bcrypt, no hardcoded secrets)
>
> ### 📊 Test Results
> All 23 tests passing ✅
> - Auth controller: 8/8 ✅
> - Middleware: 6/6 ✅
> - Routes: 5/5 ✅
> - Integration: 4/4 ✅
>
> **Recommendation:** Approved and ready to merge! 🎉

---

### Step 5: Human Review and Merge

Developer reviews PR #45:
- Checks Marty's review findings
- Verifies the code
- Maybe implements the optional recommendations
- Approves and merges

**Result:** Issue #123 is automatically closed when PR #45 merges (due to `Closes #123` in PR description)

---

## ⚙️ Required Configuration

### 1. GitHub Secrets

Add these to your repository settings:

| Secret | Description | Example |
|--------|-------------|---------|
| `MARTY_APP_ID` | Marty GitHub App ID | `2883066` |
| `MARTY_APP_PRIVATE_KEY` | Marty GitHub App private key | Download from GitHub App settings |
| `ZAI_API_KEY` | Z.ai API key | Your Z.ai platform key |

### 2. GitHub App Permissions

The Marty GitHub App must have these permissions:

| Permission | Level | Why |
|------------|-------|-----|
| Contents | Read/Write | Commit code, create branches |
| Pull requests | Read/Write | Create and comment on PRs |
| Issues | Read/Write | Comment on issues, add labels |
| Metadata | Read | Required for all apps |

### 3. Install Marty on Repository

1. Go to https://github.com/apps/marty
2. Click "Install"
3. Select your repository/organization
4. Grant necessary permissions

---

## 🚧 Advanced Configuration

### Timeout Management

Long implementations may exceed default timeout:

```yaml
- name: Marty Implements the Issue
  timeout-minutes: 45  # Adjust based on your needs
```

**Guidelines:**
- Simple bug fixes: 10-15 minutes
- Medium features: 20-30 minutes
- Complex features: 30-45 minutes
- Very complex: Consider breaking into sub-tasks

### Complexity-Based Implementation

Only implement if complexity is manageable:

```yaml
jobs:
  estimate-complexity:
    # Quick analysis job
    steps:
      - uses: nesalia-inc/marty-action@v1.0.0
        with:
          prompt: |
            Estimate complexity (1-5) of this issue.
            Output as workflow output.
        outputs:
          complexity: ${{ steps.analysis.outputs.complexity }}

  implement:
    needs: estimate-complexity
    if: needs.estimate-complexity.outputs.complexity <= 3
    # Only implement if complexity <= 3
```

### Branch Protection

Protect your main branch:

```yaml
# In GitHub repository settings
✅ Require pull request reviews before merging
✅ Require status checks to pass before merging
✅ Require branches to be up to date before merging
```

This ensures Marty's PRs are reviewed before merging.

### Cost Management

Z.ai costs can add up with long implementations:

**Strategies:**
1. **Start simple** - Begin with trivial/simple issues
2. **Use Haiku model** for quick analysis (glm-4.5-air)
3. **Limit max-turns** - Lower limits for simpler tasks
4. **Review first** - Manual review before auto-implementation for complex features

```yaml
claude_args: |
  --max-turns 15  # Lower for simple tasks
  --model glm-4.5-air  # Use faster/cheaper model when appropriate
```

---

## 🐛 Troubleshooting

### Issue: "Claude Code is not installed on this repository"

**Cause:** Marty GitHub App not installed on repository

**Solution:** Install the Marty GitHub App on the repository

### Issue: "Permission denied (publickey)"

**Cause:** Git push fails due to authentication

**Solution:** Ensure Marty's token has `contents: write` permission

### Issue: Implementation times out

**Cause:** Task too complex or `max-turns` too low

**Solution:**
- Increase `timeout-minutes`
- Increase `max-turns`
- Or break down the issue into smaller sub-tasks

### Issue: Marty creates invalid code

**Cause:** Unclear requirements or insufficient context

**Solution:**
- Improve issue triage prompt
- Ask more clarifying questions
- Provide better examples in issue description
- Manual review before requesting implementation

### Issue: PR tests are failing

**Cause:** Marty didn't run tests or implementation has bugs

**Solution:**
- Add explicit test-running step in prompt
- Ensure test command is in prompt
- Review test failures and adjust code
- Comment on PR with issues found

---

## 🎯 Best Practices

### 1. Clear Issue Descriptions

The better the issue, the better the implementation:

```markdown
## Feature: JWT Authentication

### Requirements
- JWT-based auth for API
- 24-hour token expiration
- Login endpoint: POST /api/auth/login
- Verify endpoint: GET /api/auth/verify

### Technical Details
- Users stored in PostgreSQL
- Password hashing with bcrypt
- JWT secret in environment variables

### Protected Routes
- All /api/* except /api/auth/*

### Success Criteria
- [ ] Login returns valid JWT
- [ ] Verify endpoint validates tokens
- [ ] Protected routes reject unauthenticated requests
- [ ] Unit tests included
- [ ] Error handling implemented
```

### 2. Progressive Complexity

Start with simple issues to build trust:

```
Week 1: Trivial bug fixes
Week 2: Simple features
Week 3: Medium complexity
Week 4: Complex features
```

### 3. Human-in-the-Loop

Always have human review:

```
Marty implements
    ↓
Marty self-reviews
    ↓
Human reviews (REQUIRED)
    ↓
Merge
```

### 4. Iterative Refinement

If Marty doesn't get it right:

```markdown
Comment: Almost there! A few changes needed:
1. The JWT secret should come from process.env.JWT_SECRET
2. Add rate limiting on the login endpoint
3. Update the error message to be more user-friendly

Can you update the PR?
```

Marty can iterate on the same PR/branch.

---

## 📊 Monitoring and Analytics

### Track Marty's Performance

Create a dashboard to monitor:

| Metric | How to Track |
|--------|--------------|
| Issues implemented | Count of closed issues by Marty |
| Average implementation time | Workflow duration logs |
| PR acceptance rate | Merged vs. rejected PRs |
| Review comments per PR | Average comment count |
| Test pass rate | PR test results |

### Continuous Improvement

Regularly review:
- Which issues Marty handles well
- Which types need more clarification
- Common patterns in Marty's code
- Feedback from developers

---

## 🔐 Security Considerations

### 1. Secret Management

Marty should NEVER:
- Commit hardcoded secrets
- Log sensitive data
- Expose credentials in error messages

### 2. Input Validation

Always implement:
- Request body validation
- SQL injection prevention
- XSS protection
- CSRF protection (if applicable)

### 3. Access Control

Ensure Marty's implementation:
- Respects existing authorization
- Doesn't bypass security checks
- Follows principle of least privilege

### 4. Code Review

ALWAYS have human security review for:
- Authentication changes
- Authorization changes
- Database schema changes
- External API integrations

---

## 🎓 Tips for Success

### Start Small

Don't begin with complex multi-file refactors. Start with:
- Simple bug fixes
- Adding unit tests
- Documentation updates
- Small feature additions

### Provide Context

The more context, the better:
- Link to similar code
- Provide examples
- Explain architecture decisions
- Share relevant documentation

### Iterate

First implementation might not be perfect:
- Review the code
- Provide feedback
- Ask Marty to refine
- Repeat until satisfied

### Build Trust

Start with low-risk tasks:
- Test environment first
- Non-production codebases
- Features with easy rollback
- Well-tested areas of codebase

---

## 🚀 Next Steps

1. **Setup**: Create all 4 workflow files in your repository
2. **Configure**: Add required secrets and install GitHub App
3. **Test**: Try with a simple bug fix first
4. **Iterate**: Gradually increase complexity
5. **Monitor**: Track metrics and improve prompts
6. **Scale**: Expand to more repositories

---

## 📚 Related Documentation

- [PR Review Workflow](./PR_REVIEW_WORKFLOW.md) - Detailed PR review setup
- [Marty Setup Documentation](./MARTY_SETUP_DOCUMENTATION.md) - Complete setup reference
- [Solutions Guide](./docs/solutions.md) - Automation patterns and examples
- [Custom Automations](./docs/custom-automations.md) - Advanced configuration
- [Configuration Reference](./docs/configuration.md) - All available inputs and settings

---

## 🎉 Conclusion

With these 4 workflows, you can create a fully autonomous development workflow where Marty:

1. ✅ Triages and clarifies new issues
2. ✅ Engages in technical discussion
3. ✅ Implements features automatically
4. ✅ Creates pull requests
5. ✅ Self-reviews code
6. ✅ Awaits human approval

This is the future of software development - where developers focus on high-level decisions and AI handles implementation details. 🚀

**Ready to transform your development workflow?** Start with simple issues and scale up as Marty learns your codebase!
