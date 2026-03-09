# Exercise 2.2: Pull Request Workflows & Branch Policies

## Objective
Implement PR workflows with branch policies (Azure DevOps) and branch protection rules (GitHub).

## Skills Measured
- Design and implement a pull request workflow by using branch policies and branch protections
- Implement branch merging restrictions by using branch policies and branch protections

## Prerequisites
- **Git** installed (`git --version` to verify)
- **GitHub CLI (`gh`)** installed and authenticated — the official command-line tool for GitHub. Used here to create PRs, review them, and merge from the terminal.
  - Install: https://cli.github.com/
  - Authenticate: `gh auth login`
  - Verify: `gh --version`

## Steps

### Part A: GitHub Branch Protection Rules

1. **Configure branch protection** for `main`:
   - Repository → Settings → Branches → Add rule
   - Branch name pattern: `main`
   - Enable:
     - [x] Require a pull request before merging
       - Required approving reviews: 1
       - Dismiss stale PR reviews when new commits are pushed
       - Require review from code owners
     - [x] Require status checks to pass before merging
       - Add required checks: "build", "test"
       - Require branches to be up to date before merging
     - [x] Require conversation resolution before merging
     - [x] Require signed commits (optional, but know it for the exam)
     - [x] Require linear history (forces squash or rebase merge)
     - [x] Include administrators (enforce for everyone)

2. **Create a CODEOWNERS file** (`.github/CODEOWNERS`):
   ```
   # Default owners for everything
   * @your-username

   # Backend code owners
   /src/api/ @backend-team
   /src/database/ @backend-team @dba-team

   # Frontend code owners
   /src/ui/ @frontend-team

   # Pipeline code owners
   /.github/workflows/ @devops-team
   /cicd/ @devops-team

   # Infrastructure code owners
   /infra/ @platform-team
   ```

3. **Create a PR template** (`.github/pull_request_template.md`):
   ```markdown
   ## Description
   <!-- Describe your changes -->

   ## Type of Change
   - [ ] Bug fix
   - [ ] New feature
   - [ ] Breaking change
   - [ ] Documentation update

   ## Checklist
   - [ ] Tests pass locally
   - [ ] Code follows style guidelines
   - [ ] Self-reviewed
   - [ ] Documentation updated
   - [ ] No sensitive data in code

   ## Related Issues
   Closes #

   ## Screenshots (if applicable)
   ```

4. **Test the protection rules**:
   ```bash
   # Try to push directly to main (should fail)
   git checkout main
   echo "test" > test.txt
   git add . && git commit -m "test: direct push"
   git push origin main  # Should be rejected!

   # Proper flow via PR
   git checkout -b feature/test-protection
   git push origin feature/test-protection
   gh pr create --title "Test branch protection" --body "Testing protections"
   ```

### Part B: Azure DevOps Branch Policies

1. **Configure branch policies** in Azure DevOps:
   - Repos → Branches → main → ⋯ → Branch policies

2. **Set up these policies**:
   - **Require a minimum number of reviewers**: 1
     - Allow requestors to approve their own changes: No
     - Require re-approval when changes are pushed: Yes
   - **Check for linked work items**: Required
   - **Check for comment resolution**: Required
   - **Limit merge types**: Allow only Squash merge
   - **Build validation**:
     - Add build policy
     - Select your CI pipeline
     - Trigger: Automatic
     - Policy requirement: Required
     - Build expiration: 12 hours

3. **Add path-based policies**:
   - Path filter: `/src/api/*` → Require additional reviewer from API team
   - Path filter: `/infra/*` → Require reviewer from Platform team

4. **Configure automatic reviewers**:
   - Add: Platform team as reviewer when `/infra/*` files change
   - Add: Security team as reviewer when `**/security*` files change

### Part C: Merge Strategies

1. **Practice different merge types**:

   ```bash
   # Setup
   git checkout main
   git checkout -b feature/merge-demo
   echo "feature work" > feature.txt
   git add . && git commit -m "feat: add feature"
   echo "more work" >> feature.txt
   git add . && git commit -m "feat: extend feature"
   ```

   **Merge commit** (preserves full history):
   ```bash
   git checkout main
   git merge feature/merge-demo --no-ff
   # Creates a merge commit, preserves branch history
   git log --oneline --graph
   ```

   **Squash merge** (compressed history):
   ```bash
   git checkout main
   git merge feature/merge-demo --squash
   git commit -m "feat: add feature (squashed)"
   # All commits compressed into one
   ```

   **Rebase merge** (linear history):
   ```bash
   git checkout feature/merge-demo
   git rebase main
   git checkout main
   git merge feature/merge-demo --ff-only
   # Linear history, no merge commit
   ```

2. **Document when to use each**:
   | Strategy | History | Best For |
   |----------|---------|----------|
   | Merge commit | Complete with branches | Open source, auditability |
   | Squash merge | Clean, linear | Feature branches, clean main |
   | Rebase merge | Linear, individual commits | Small changes, linear history |

### Part D: PR Automation with GitHub Actions

1. **Auto-assign reviewers** (`.github/workflows/pr-automation.yml`):
   ```yaml
   name: PR Automation
   on:
     pull_request:
       types: [opened, ready_for_review]

   jobs:
     auto-label:
       runs-on: ubuntu-latest
       permissions:
         pull-requests: write
       steps:
         - uses: actions/labeler@v5
           with:
             repo-token: "${{ secrets.GITHUB_TOKEN }}"
   ```

2. **Create labeler config** (`.github/labeler.yml`):
   ```yaml
   backend:
     - changed-files:
       - any-glob-to-any-file: 'src/api/**'
   frontend:
     - changed-files:
       - any-glob-to-any-file: 'src/ui/**'
   infrastructure:
     - changed-files:
       - any-glob-to-any-file: 'infra/**'
   pipeline:
     - changed-files:
       - any-glob-to-any-file: '.github/workflows/**'
   ```

## Validation Checklist

- [ ] GitHub branch protection rules configured and tested
- [ ] CODEOWNERS file created and working
- [ ] PR template created
- [ ] Direct push to protected branch rejected
- [ ] Azure DevOps branch policies configured (reviewers, build validation, work item linking)
- [ ] Practiced all three merge strategies (merge, squash, rebase)
- [ ] PR automation with labels configured

## Key Concepts to Review

- **Branch policies vs branch protections**: Azure DevOps vs GitHub terminology
- **CODEOWNERS**: Automatic reviewer assignment based on file paths
- **Squash vs Merge vs Rebase**: When and why to use each
- **Build validation**: CI must pass before PR can merge

## Microsoft Learn References
- [Branch policies](https://learn.microsoft.com/en-us/azure/devops/repos/git/branch-policies)
- [GitHub branch protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-a-branch-protection-rule)
