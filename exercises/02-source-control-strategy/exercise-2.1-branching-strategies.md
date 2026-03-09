# Exercise 2.1: Branching Strategies

## Objective
Understand, compare, and implement different branching strategies: trunk-based development, feature branching, and release branching.

## Skills Measured
- Design a branch strategy, including trunk-based, feature branch, and release branch

## Prerequisites
- **Git** installed (2.0+ recommended) — the version control system. Used extensively for branching, merging, and tagging.
  - Verify: `git --version`

## Steps

### Part A: Trunk-Based Development

Trunk-based development keeps all development on `main` (trunk) with short-lived feature branches (< 1 day).

1. **Set up the workflow**:
   ```bash
   # Create a new repo to practice
   mkdir branching-demo && cd branching-demo
   git init
   echo "# Branching Demo" > README.md
   git add . && git commit -m "initial commit"
   ```

2. **Simulate trunk-based flow**:
   ```bash
   # Developer A: Short-lived branch (< 1 day)
   git checkout -b feat/add-login
   echo "login feature" > login.js
   git add . && git commit -m "feat: add login"
   git checkout main && git merge feat/add-login --no-ff
   git branch -d feat/add-login

   # Developer B: Short-lived branch
   git checkout -b feat/add-auth
   echo "auth middleware" > auth.js
   git add . && git commit -m "feat: add auth"
   git checkout main && git merge feat/add-auth --no-ff
   git branch -d feat/add-auth
   ```

3. **Key characteristics to document**:
   - Branches live < 1 day
   - Frequent merges to main
   - Feature flags used to hide incomplete work
   - Main is always deployable
   - Best for: CI/CD mature teams, small teams

### Part B: Feature Branch Workflow

Feature branches live longer and are merged via PRs.

1. **Simulate feature branch flow**:
   ```bash
   # Feature branch with longer lifecycle
   git checkout -b feature/user-dashboard

   # Multiple commits over days
   echo "dashboard layout" > dashboard.js
   git add . && git commit -m "feat: dashboard layout"
   echo "dashboard api" > dashboard-api.js
   git add . && git commit -m "feat: dashboard API integration"
   echo "dashboard tests" > dashboard.test.js
   git add . && git commit -m "test: add dashboard tests"

   # Merge via PR (simulated)
   git checkout main
   git merge feature/user-dashboard --no-ff
   git branch -d feature/user-dashboard
   ```

2. **Key characteristics**:
   - Branches live days to weeks
   - PR reviews required before merge
   - CI runs on feature branches
   - Best for: Medium teams, projects needing code review

### Part C: Release Branch (GitFlow) Strategy

1. **Implement GitFlow**:
   ```bash
   # Main branch = production
   # Develop branch = integration
   git checkout -b develop

   # Feature from develop
   git checkout -b feature/payments develop
   echo "payments" > payments.js
   git add . && git commit -m "feat: add payments"
   git checkout develop && git merge feature/payments --no-ff

   # Create release branch
   git checkout -b release/1.0.0 develop
   echo "1.0.0" > VERSION
   git add . && git commit -m "chore: bump version to 1.0.0"

   # Merge release to main AND develop
   git checkout main && git merge release/1.0.0 --no-ff
   git tag -a v1.0.0 -m "Release 1.0.0"
   git checkout develop && git merge release/1.0.0 --no-ff
   git branch -d release/1.0.0

   # Hotfix from main
   git checkout -b hotfix/1.0.1 main
   echo "fix critical bug" > hotfix.js
   git add . && git commit -m "fix: critical payment bug"
   git checkout main && git merge hotfix/1.0.1 --no-ff
   git tag -a v1.0.1 -m "Hotfix 1.0.1"
   git checkout develop && git merge hotfix/1.0.1 --no-ff
   git branch -d hotfix/1.0.1
   ```

2. **Key characteristics**:
   - Multiple long-lived branches (main, develop)
   - Release branches for stabilization
   - Hotfix branches from main for production fixes
   - Best for: Large teams, scheduled releases, multiple versions in production

### Part D: Comparison Matrix

Create this comparison and memorize it:

| Aspect | Trunk-Based | Feature Branch | GitFlow |
|--------|-------------|----------------|---------|
| Branch lifetime | Hours | Days-weeks | Varies |
| Merge frequency | Multiple/day | Per feature | Per release |
| Complexity | Low | Medium | High |
| Team size | Small-medium | Medium | Large |
| Release cadence | Continuous | Frequent | Scheduled |
| Feature flags needed | Yes | Optional | No |
| Code review | Pair programming | PR reviews | PR reviews |
| Risk of merge conflicts | Low | Medium | High |

## Validation Checklist

- [ ] Practiced all three branching strategies hands-on
- [ ] Can explain when to use each strategy
- [ ] Understand GitFlow hotfix and release branch flow
- [ ] Can articulate pros/cons of each approach
- [ ] Created comparison matrix

## Microsoft Learn References
- [Adopt a Git branching strategy](https://learn.microsoft.com/en-us/azure/devops/repos/git/git-branching-guidance)
