# Exercise 2.4: Repository Scaling, Permissions & Git Recovery

## Objective
Optimize large repositories, configure permissions and tags, and practice Git data recovery.

## Skills Measured
- Design a strategy for scaling and optimizing a Git repository, including Scalar and cross-repository sharing
- Configure permissions in the source control repository
- Configure tags to organize the source control repository
- Recover specific data by using Git commands
- Remove specific data from source control

## Prerequisites
- **Git 2.38+** — needed for Scalar support. Verify: `git --version`
- **Scalar** — a Microsoft tool for optimizing very large Git repositories. It enables partial clone, sparse checkout, and commit-graph features. Bundled with Git 2.38+.
  - Verify: `scalar version`
- **Python 3 & pip** — Python runtime and package installer. Needed to install git-filter-repo.
  - Install: https://www.python.org/downloads/
  - Verify: `python3 --version && pip --version`
- **git-filter-repo** — a Python tool for rewriting Git history (e.g., removing sensitive data from all commits). Replaces the older `git filter-branch`.
  - Install: `pip install git-filter-repo`
- **BFG Repo-Cleaner** (optional alternative) — a simpler tool for removing large files or secrets from Git history. Requires Java.
  - Download: https://rtyley.github.io/bfg-repo-cleaner/

## Steps

### Part A: Repository Optimization with Scalar

1. **Understand Scalar** (by Microsoft, for very large repos):
   ```bash
   # Install Scalar (included in Git 2.38+)
   # Scalar enables: partial clone, sparse checkout, commit graph, filesystem monitor

   # Register a repo with Scalar
   scalar register

   # Or clone with Scalar optimizations
   scalar clone https://github.com/large-org/large-repo
   ```

2. **Practice partial clone** (fetches objects on-demand):
   ```bash
   # Blobless clone (no file content until checkout)
   git clone --filter=blob:none https://github.com/your-org/your-repo.git

   # Treeless clone (no tree objects until needed)
   git clone --filter=tree:0 https://github.com/your-org/your-repo.git
   ```

3. **Practice sparse checkout** (only check out specific folders):
   ```bash
   git clone --no-checkout https://github.com/your-org/monorepo.git
   cd monorepo
   git sparse-checkout init --cone
   git sparse-checkout set src/my-service src/shared
   git checkout main
   # Only src/my-service and src/shared are on disk
   ```

4. **Enable commit graph** for faster log operations:
   ```bash
   git commit-graph write --reachable
   git config --global core.commitGraph true
   ```

5. **Shallow clone** for CI/CD:
   ```bash
   # Only last 1 commit (fastest for CI)
   git clone --depth 1 https://github.com/your-org/your-repo.git

   # Fetch more history if needed
   git fetch --deepen=10
   ```

### Part B: Configure Permissions

**GitHub Repository Permissions:**

1. **Repository roles** (know these for the exam):
   | Role | Read | Triage | Write | Maintain | Admin |
   |------|------|--------|-------|----------|-------|
   | View code | Yes | Yes | Yes | Yes | Yes |
   | Create issues | Yes | Yes | Yes | Yes | Yes |
   | Manage issues | - | Yes | Yes | Yes | Yes |
   | Push code | - | - | Yes | Yes | Yes |
   | Manage branches | - | - | - | Yes | Yes |
   | Admin settings | - | - | - | - | Yes |

2. **Configure team permissions**:
   - Organization → Teams → Create teams
   - Assign repository access per team
   - Use CODEOWNERS for file-level review requirements

**Azure DevOps Repository Permissions:**

1. **Security groups and permissions**:
   - Project Settings → Repositories → Security
   - Groups: Readers, Contributors, Build Administrators, Project Administrators
   - Key permissions:
     - Contribute
     - Create branch
     - Force push (deny for most users!)
     - Manage permissions
     - Edit policies

2. **Per-branch permissions**:
   - Repos → Branches → Branch security
   - Deny force push on main, release, and hotfix branches

### Part C: Git Tags

1. **Create tags for releases**:
   ```bash
   # Lightweight tag
   git tag v1.0.0

   # Annotated tag (preferred — includes metadata)
   git tag -a v1.0.0 -m "Release 1.0.0 - Initial production release"

   # Tag a specific commit
   git tag -a v0.9.0 abc1234 -m "Beta release"

   # Push tags
   git push origin v1.0.0
   git push origin --tags  # Push all tags

   # List tags
   git tag -l "v1.*"

   # Show tag details
   git show v1.0.0
   ```

2. **Semantic versioning with tags**:
   ```
   v1.0.0 - Major release
   v1.1.0 - Minor feature addition
   v1.1.1 - Patch/bugfix
   v2.0.0-beta.1 - Pre-release
   v2.0.0-rc.1 - Release candidate
   ```

3. **Create a GitHub release from a tag**:
   ```bash
   gh release create v1.0.0 --title "v1.0.0" --notes "First production release"
   ```

### Part D: Git Recovery Operations

1. **Recover deleted branch**:
   ```bash
   # Find the commit hash of the deleted branch
   git reflog
   # Look for: abc1234 HEAD@{5}: checkout: moving from deleted-branch to main

   # Recreate the branch
   git checkout -b recovered-branch abc1234
   ```

2. **Recover deleted file from history**:
   ```bash
   # Find when the file was deleted
   git log --diff-filter=D --summary -- path/to/deleted-file.js

   # Restore the file from the commit before deletion
   git checkout <commit-hash>^ -- path/to/deleted-file.js
   ```

3. **Undo a bad merge**:
   ```bash
   # Revert a merge commit (keeps history)
   git revert -m 1 <merge-commit-hash>

   # Or reset if not pushed (destructive!)
   git reset --hard HEAD~1
   ```

4. **Cherry-pick a specific commit**:
   ```bash
   # Apply a specific commit to current branch
   git cherry-pick <commit-hash>

   # Cherry-pick without committing (stage only)
   git cherry-pick --no-commit <commit-hash>
   ```

5. **Interactive rebase to fix history** (before pushing):
   ```bash
   # Edit last 3 commits
   git rebase -i HEAD~3

   # Options in the editor:
   # pick   = keep commit
   # reword = change commit message
   # edit   = stop to amend
   # squash = merge with previous
   # drop   = remove commit
   ```

### Part E: Remove Sensitive Data

1. **Remove a file from all history** using `git filter-repo`:
   ```bash
   # Install git-filter-repo
   pip install git-filter-repo

   # Remove a file from entire history
   git filter-repo --invert-paths --path secrets.json

   # Remove a folder from entire history
   git filter-repo --invert-paths --path config/credentials/
   ```

2. **Using BFG Repo-Cleaner** (alternative):
   ```bash
   # Remove files > 100MB from history
   bfg --strip-blobs-bigger-than 100M

   # Remove a specific file
   bfg --delete-files secrets.json

   # Clean up
   git reflog expire --expire=now --all && git gc --prune=now --aggressive
   ```

3. **After removing sensitive data**:
   ```bash
   # Force push rewritten history
   git push --force-with-lease

   # Rotate any exposed secrets immediately!
   # Invalidate API keys, tokens, passwords
   ```

4. **Prevention** — `.gitignore`:
   ```gitignore
   # Secrets and credentials
   *.pem
   *.key
   .env
   .env.local
   secrets.json
   credentials.yml

   # IDE files
   .vscode/settings.json
   .idea/

   # Dependencies
   node_modules/
   ```

## Validation Checklist

- [ ] Scalar concepts understood (partial clone, sparse checkout, commit graph)
- [ ] Shallow clone practiced
- [ ] GitHub repository permissions understood (roles matrix)
- [ ] Azure DevOps repository permissions configured
- [ ] Annotated tags created and pushed
- [ ] Semantic versioning scheme understood
- [ ] Git reflog used to recover deleted branch
- [ ] Deleted file recovered from Git history
- [ ] git filter-repo or BFG used to remove sensitive data
- [ ] .gitignore configured for security

## Microsoft Learn References
- [Git Scalar](https://learn.microsoft.com/en-us/azure/devops/repos/git/scalar)
- [Set repository permissions](https://learn.microsoft.com/en-us/azure/devops/repos/git/set-git-repository-permissions)
- [Remove large binaries from history](https://learn.microsoft.com/en-us/azure/devops/repos/git/remove-binaries)
