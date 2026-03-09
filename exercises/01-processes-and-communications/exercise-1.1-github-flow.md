# Exercise 1.1: GitHub Flow & Work Tracking

## Objective
Implement GitHub Flow branching strategy with full work item traceability using GitHub Issues and GitHub Projects.

## Skills Measured
- Design and implement a structure for the flow of work, including GitHub Flow
- Design and implement a strategy for feedback cycles, including notifications and GitHub issues
- Design and implement integration for tracking work, including GitHub projects

## Prerequisites
- GitHub account with a repository (you can use this repo)
- **GitHub CLI (`gh`)** installed and authenticated. The GitHub CLI is the official command-line tool for GitHub — it lets you create repositories, issues, pull requests, labels, and more from the terminal without opening the browser.
  - Install: https://cli.github.com/
  - Authenticate: run `gh auth login` and follow the prompts
  - Verify: `gh --version`
- **Git** installed (`git --version` to verify)

## Steps

### Part A: Set Up GitHub Issues for Work Tracking

1. **Create a GitHub repository** (or use an existing one):
   ```bash
   gh repo create az400-exercises --public --clone
   cd az400-exercises
   ```

2. **Create Issue Templates** — Create `.github/ISSUE_TEMPLATE/`:

   **Bug Report** (`.github/ISSUE_TEMPLATE/bug_report.md`):
   ```markdown
   ---
   name: Bug Report
   about: Report a bug
   title: '[BUG] '
   labels: bug
   assignees: ''
   ---

   ## Description
   A clear description of the bug.

   ## Steps to Reproduce
   1.
   2.
   3.

   ## Expected Behavior

   ## Actual Behavior

   ## Environment
   - OS:
   - Browser:
   - Version:
   ```

   **Feature Request** (`.github/ISSUE_TEMPLATE/feature_request.md`):
   ```markdown
   ---
   name: Feature Request
   about: Suggest a new feature
   title: '[FEATURE] '
   labels: enhancement
   assignees: ''
   ---

   ## Problem Statement

   ## Proposed Solution

   ## Acceptance Criteria
   - [ ]
   - [ ]
   ```

3. **Create labels** for tracking:
   ```bash
   gh label create "priority:high" --color "B60205" --description "High priority"
   gh label create "priority:medium" --color "FBCA04" --description "Medium priority"
   gh label create "priority:low" --color "0E8A16" --description "Low priority"
   gh label create "status:in-progress" --color "1D76DB" --description "Work in progress"
   gh label create "status:review" --color "5319E7" --description "Ready for review"
   ```

4. **Create several issues** to work with:
   ```bash
   gh issue create --title "[FEATURE] Add health check endpoint" --body "Add /health endpoint" --label "enhancement,priority:high"
   gh issue create --title "[FEATURE] Add logging middleware" --body "Add structured logging" --label "enhancement,priority:medium"
   gh issue create --title "[BUG] Server crashes on invalid input" --body "No input validation" --label "bug,priority:high"
   ```

### Part B: Set Up GitHub Projects (v2)

1. **Create a GitHub Project** (board view):
   - Go to your profile → Projects → New Project
   - Choose "Board" template
   - Name it "AZ-400 Sprint Board"

2. **Configure columns**: Todo, In Progress, In Review, Done

3. **Add custom fields**:
   - `Priority` (Single select: High, Medium, Low)
   - `Sprint` (Iteration field)
   - `Story Points` (Number)
   - `Type` (Single select: Feature, Bug, Tech Debt)

4. **Add your issues** to the project board

5. **Create views**:
   - Board view (default Kanban)
   - Table view (for sprint planning)
   - Create a filter view showing only high-priority items

### Part C: Implement GitHub Flow

1. **Understand GitHub Flow**:
   - `main` is always deployable
   - Create feature branches from `main`
   - Open PR, review, merge back to `main`

2. **Work on Issue #1** (health check endpoint):
   ```bash
   # Create a feature branch (reference the issue number)
   git checkout -b feature/1-health-check

   # Make changes — add health check to hello.js
   ```

3. **Add health check endpoint** to `hello.js`:
   ```javascript
   const server = http.createServer((req, res) => {
     if (req.url === '/health') {
       res.statusCode = 200;
       res.setHeader('Content-Type', 'application/json');
       res.end(JSON.stringify({ status: 'healthy', timestamp: new Date().toISOString() }));
       return;
     }
     res.statusCode = 200;
     res.setHeader('Content-Type', 'text/plain');
     res.end('Hello, World!\n');
   });
   ```

4. **Commit with conventional commit message** referencing the issue:
   ```bash
   git add .
   git commit -m "feat: add health check endpoint

   Closes #1"
   ```

5. **Push and create a Pull Request**:
   ```bash
   git push origin feature/1-health-check
   gh pr create --title "feat: add health check endpoint" --body "Closes #1" --assignee @me
   ```

6. **Observe**: The PR should auto-link to Issue #1 and the project board should update.

7. **Merge the PR**:
   ```bash
   gh pr merge --squash --delete-branch
   ```

8. **Verify** that Issue #1 is automatically closed and moved to "Done" on the project board.

### Part D: Configure Notifications

1. Go to GitHub → Settings → Notifications
2. Configure notification preferences for:
   - Participating conversations
   - Watching repositories
   - GitHub Actions workflow runs
3. Set up email notifications for failed CI runs

## Validation Checklist

- [ ] Issue templates are created and working
- [ ] Custom labels are configured
- [ ] GitHub Project board has custom fields and multiple views
- [ ] Feature branch was created, PR opened, and merged using GitHub Flow
- [ ] Issue was auto-closed via PR merge
- [ ] Project board reflects the current state of work items
- [ ] Notifications are configured

## Key Concepts to Review

- **GitHub Flow vs GitFlow vs Trunk-Based**: Know when to use each
- **Linking PRs to Issues**: Using keywords (Closes, Fixes, Resolves)
- **GitHub Projects v2**: Custom fields, views, workflows, insights
- **Conventional Commits**: Structured commit messages for automation

## Microsoft Learn References
- [About GitHub Flow](https://docs.github.com/en/get-started/quickstart/github-flow)
- [About GitHub Projects](https://docs.github.com/en/issues/planning-and-tracking-with-projects)
- [Linking PRs to Issues](https://docs.github.com/en/issues/tracking-your-work-with-issues/linking-a-pull-request-to-an-issue)
