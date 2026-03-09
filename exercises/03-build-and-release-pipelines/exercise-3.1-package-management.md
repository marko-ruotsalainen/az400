# Exercise 3.1: Package Management

## Objective
Configure package feeds using GitHub Packages and Azure Artifacts, implement dependency versioning strategies.

## Skills Measured
- Recommend package management tools including GitHub Packages registry and Azure Artifacts
- Design and implement package feeds and views for local and upstream packages
- Design and implement a dependency versioning strategy (SemVer and CalVer)
- Design and implement a versioning strategy for pipeline artifacts

## Prerequisites
- **Node.js & npm** — JavaScript runtime and package manager. Used to create, publish, and consume packages.
  - Install: https://nodejs.org/ (LTS version, includes npm)
  - Verify: `node --version && npm --version`
- **Azure CLI (`az`)** — used for publishing universal packages to Azure Artifacts.
  - Install: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli
  - Authenticate: `az login`
- GitHub account and Azure DevOps organization from previous exercises

## Steps

### Part A: GitHub Packages

1. **Publish an npm package to GitHub Packages**:

   Create `packages/my-utils/package.json`:
   ```json
   {
     "name": "@YOUR_USERNAME/my-utils",
     "version": "1.0.0",
     "description": "Shared utilities",
     "main": "index.js",
     "publishConfig": {
       "registry": "https://npm.pkg.github.com"
     }
   }
   ```

   Create `packages/my-utils/index.js`:
   ```javascript
   module.exports = {
     formatDate: (date) => date.toISOString().split('T')[0],
     generateId: () => Math.random().toString(36).substring(2, 9),
   };
   ```

2. **Authenticate and publish**:
   ```bash
   # Create .npmrc for GitHub Packages
   echo "@YOUR_USERNAME:registry=https://npm.pkg.github.com" > .npmrc
   echo "//npm.pkg.github.com/:_authToken=\${GITHUB_TOKEN}" >> .npmrc

   # Publish
   cd packages/my-utils
   npm publish
   ```

3. **Publish via GitHub Actions**:
   ```yaml
   name: Publish Package
   on:
     release:
       types: [created]

   jobs:
     publish:
       runs-on: ubuntu-latest
       permissions:
         packages: write
         contents: read
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-node@v4
           with:
             node-version: 20
             registry-url: https://npm.pkg.github.com/
         - run: npm publish
           working-directory: packages/my-utils
           env:
             NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
   ```

### Part B: Azure Artifacts

1. **Create an Azure Artifacts feed**:
   - Azure DevOps → Artifacts → Create Feed
   - Name: "az400-packages"
   - Visibility: Organization-scoped
   - Upstream sources: Enable npm, NuGet, Maven, Python

2. **Configure upstream sources**:
   - npmjs.com (public npm registry)
   - nuget.org (public NuGet)
   - This allows the feed to cache packages from public registries

3. **Create feed views**:
   - `@Local` (default) — All packages
   - `@Prerelease` — Pre-release packages
   - `@Release` — Stable, production-ready packages

4. **Promote packages between views**:
   ```bash
   # Using Azure CLI
   az artifacts universal publish \
     --organization https://dev.azure.com/YOUR_ORG \
     --project AZ400-Prep \
     --scope project \
     --feed az400-packages \
     --name my-package \
     --version 1.0.0 \
     --path .
   ```

5. **Consume from Azure Artifacts in a pipeline**:
   ```yaml
   steps:
     - task: npmAuthenticate@0
       inputs:
         workingFile: .npmrc
     - script: npm install
       displayName: 'Install from Azure Artifacts feed'
   ```

### Part C: Versioning Strategies

1. **Semantic Versioning (SemVer)**:
   ```
   MAJOR.MINOR.PATCH
   1.0.0 - Initial release
   1.1.0 - New feature (backward compatible)
   1.1.1 - Bug fix
   2.0.0 - Breaking change
   2.0.0-beta.1 - Pre-release
   2.0.0-rc.1 - Release candidate
   ```

2. **Calendar Versioning (CalVer)**:
   ```
   YYYY.MM.DD or YYYY.MM.MICRO
   2026.03.08 - Release on March 8, 2026
   2026.03.1 - First March 2026 release
   ```

3. **Pipeline artifact versioning**:
   ```yaml
   # Azure Pipelines — version with build number
   variables:
     Major: '1'
     Minor: '0'
     Patch: $[counter(format('{0}.{1}', variables['Major'], variables['Minor']), 0)]

   name: $(Major).$(Minor).$(Patch)

   steps:
     - script: echo "##vso[build.updatebuildnumber]$(Major).$(Minor).$(Patch)"
   ```

   ```yaml
   # GitHub Actions — version with run number
   env:
     VERSION: 1.0.${{ github.run_number }}
   ```

4. **Implement SemVer automation** in your pipeline:
   ```yaml
   # Use GitVersion for automatic SemVer from Git history
   steps:
     - task: gitversion/setup@0
       with:
         versionSpec: '5.x'
     - task: gitversion/execute@0
     - script: |
         echo "SemVer: $(GitVersion.SemVer)"
         echo "Full: $(GitVersion.FullSemVer)"
         echo "NuGet: $(GitVersion.NuGetVersion)"
   ```

### Part D: Comparison Matrix

| Feature | GitHub Packages | Azure Artifacts |
|---------|----------------|-----------------|
| Supported formats | npm, Docker, Maven, NuGet, RubyGems | npm, NuGet, Maven, Python, Universal |
| Feed views | N/A | @Local, @Prerelease, @Release |
| Upstream sources | N/A | npm, NuGet, Maven, Python public registries |
| Scope | Repository or Organization | Project or Organization |
| Free tier | 500MB storage | 2GB per organization |
| Access control | GitHub permissions | Azure DevOps permissions |

## Validation Checklist

- [ ] npm package published to GitHub Packages
- [ ] GitHub Actions workflow for package publishing created
- [ ] Azure Artifacts feed created with upstream sources
- [ ] Feed views configured (@Local, @Prerelease, @Release)
- [ ] Package consumed from feed in a pipeline
- [ ] SemVer and CalVer differences understood
- [ ] Pipeline artifact versioning implemented
- [ ] Comparison matrix memorized

## Microsoft Learn References
- [GitHub Packages](https://docs.github.com/en/packages)
- [Azure Artifacts overview](https://learn.microsoft.com/en-us/azure/devops/artifacts/start-using-azure-artifacts)
- [Feed views](https://learn.microsoft.com/en-us/azure/devops/artifacts/concepts/views)
