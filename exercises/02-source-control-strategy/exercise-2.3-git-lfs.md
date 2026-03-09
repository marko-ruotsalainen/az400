# Exercise 2.3: Git LFS & Large File Management

## Objective
Implement Git LFS for managing large files and understand strategies for handling binary assets.

## Skills Measured
- Design and implement a strategy for managing large files, including Git Large File Storage (LFS) and git-fat

## Prerequisites
- **Git** installed (`git --version` to verify)
- **Git LFS** — an extension for Git that handles large files (images, videos, datasets) by storing them outside the main repo. Git LFS replaces large files with lightweight pointer files.
  - Install: https://git-lfs.com/ or `sudo apt install git-lfs` / `brew install git-lfs`
  - Initialize: `git lfs install`
  - Verify: `git lfs version`
- **GitHub CLI (`gh`)** installed and authenticated (see Exercise 1.1 for setup)

## Steps

### Part A: Set Up Git LFS

1. **Install Git LFS**:
   ```bash
   # Ubuntu/Debian
   sudo apt-get install git-lfs

   # macOS
   brew install git-lfs

   # Initialize LFS in your repo
   cd az400-exercises
   git lfs install
   ```

2. **Track large file types**:
   ```bash
   # Track common binary/large file types
   git lfs track "*.zip"
   git lfs track "*.tar.gz"
   git lfs track "*.jar"
   git lfs track "*.dll"
   git lfs track "*.exe"
   git lfs track "*.png"
   git lfs track "*.jpg"
   git lfs track "*.mp4"
   git lfs track "*.psd"
   git lfs track "*.ai"

   # Verify tracking rules
   cat .gitattributes
   ```

3. **Commit the `.gitattributes` file**:
   ```bash
   git add .gitattributes
   git commit -m "chore: configure Git LFS tracking"
   ```

4. **Test LFS with a large file**:
   ```bash
   # Create a sample large file
   dd if=/dev/zero of=large-binary.zip bs=1M count=50
   git add large-binary.zip
   git commit -m "chore: add large binary for testing"
   git push

   # Verify the file is tracked by LFS
   git lfs ls-files
   ```

5. **Examine the pointer file**:
   ```bash
   # See what Git actually stores (a pointer, not the binary)
   git show HEAD:large-binary.zip
   # Output will be an LFS pointer:
   # version https://git-lfs.github.com/spec/v1
   # oid sha256:...
   # size 52428800
   ```

### Part B: LFS in CI/CD Pipelines

1. **Azure Pipelines with LFS**:
   ```yaml
   steps:
     - checkout: self
       lfs: true  # Enable LFS checkout
       fetchDepth: 1  # Shallow clone + LFS
   ```

2. **GitHub Actions with LFS**:
   ```yaml
   steps:
     - uses: actions/checkout@v4
       with:
         lfs: true  # Fetch LFS files
   ```

3. **Optimize LFS in CI** — only fetch needed files:
   ```bash
   # Skip LFS during clone, fetch specific files later
   GIT_LFS_SKIP_SMUDGE=1 git clone <repo>
   git lfs pull --include="path/to/needed/files/*"
   ```

### Part C: LFS Migration

1. **Migrate existing history** to LFS:
   ```bash
   # Migrate all .zip files in history to LFS
   git lfs migrate import --include="*.zip" --everything

   # Verify migration
   git lfs ls-files

   # Now force push (this rewrites history!)
   # git push --force-with-lease  # Only if team is aware!
   ```

2. **Check LFS storage usage**:
   ```bash
   # GitHub LFS storage
   gh api /repos/{owner}/{repo} --jq '.size'

   # List all LFS objects
   git lfs ls-files --long
   ```

### Part D: Understand Alternatives

Study these alternatives (know them for the exam):

| Tool | Use Case | How It Works |
|------|----------|-------------|
| **Git LFS** | Binary files in Git repos | Pointer files in Git, binary in LFS store |
| **git-fat** | Large files (simpler than LFS) | Stores in external object store (S3, etc.) |
| **Azure Artifacts** | Build outputs, packages | Package management, not version-controlled |
| **Git Scalar** | Very large repos (monorepos) | Partial clone + sparse checkout |

### Part E: .gitattributes Best Practices

```gitattributes
# Git LFS tracking
*.zip filter=lfs diff=lfs merge=lfs -text
*.tar.gz filter=lfs diff=lfs merge=lfs -text
*.png filter=lfs diff=lfs merge=lfs -text

# Ensure consistent line endings
*.js text eol=lf
*.ts text eol=lf
*.json text eol=lf
*.yaml text eol=lf
*.yml text eol=lf
*.md text eol=lf

# Binary files
*.ico binary
*.woff binary
*.woff2 binary
```

## Validation Checklist

- [ ] Git LFS installed and initialized
- [ ] File types tracked in `.gitattributes`
- [ ] Large file committed and verified in LFS
- [ ] LFS pointer file structure understood
- [ ] LFS checkout configured in Azure Pipeline and GitHub Actions
- [ ] LFS migration command practiced
- [ ] Can explain LFS vs git-fat vs Azure Artifacts

## Microsoft Learn References
- [Manage large files with Git LFS](https://learn.microsoft.com/en-us/azure/devops/repos/git/manage-large-files)
