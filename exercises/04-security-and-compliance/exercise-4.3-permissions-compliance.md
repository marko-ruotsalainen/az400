# Exercise 4.3: Permissions, Access Control & Compliance

## Objective
Configure permissions and roles in GitHub and Azure DevOps, including access levels and project/team configuration.

## Skills Measured
- Design and implement permissions and roles in GitHub
- Design and implement permissions and security groups in Azure DevOps
- Recommend appropriate access levels (stakeholder access, outside collaborator)
- Configure projects and teams in Azure DevOps

## Steps

### Part A: GitHub Permissions

1. **Organization roles**:
   | Role | Capabilities |
   |------|-------------|
   | Owner | Full admin, billing, can delete org |
   | Member | Create repos, see other members |
   | Outside Collaborator | Access specific repos only |
   | Billing Manager | Manage billing only |

2. **Repository roles**:
   | Role | Read | Triage | Write | Maintain | Admin |
   |------|------|--------|-------|----------|-------|
   | Clone/Pull | Yes | Yes | Yes | Yes | Yes |
   | Manage issues | - | Yes | Yes | Yes | Yes |
   | Push | - | - | Yes | Yes | Yes |
   | Manage branches | - | - | - | Yes | Yes |
   | Settings/Delete | - | - | - | - | Yes |

3. **Configure team-based permissions**:
   - Create teams: `backend-devs`, `frontend-devs`, `devops`, `security`
   - Assign repository access per team
   - Use nested teams for inheritance

4. **Outside collaborators** (exam topic!):
   - External users with access to specific repositories
   - Cannot see organization members or other repos
   - Use for: contractors, open-source contributors
   - Audit: Settings → People → Outside collaborators

### Part B: Azure DevOps Permissions

1. **Access levels** (key exam topic!):
   | Access Level | Cost | Capabilities |
   |---|---|---|
   | **Stakeholder** | Free (unlimited) | View boards, create work items, view pipelines |
   | **Basic** | Included (5 free) | Full Boards, Repos, Pipelines, Test Plans (limited) |
   | **Basic + Test Plans** | Paid | Full Test Plans capabilities |
   | **Visual Studio subscriber** | With VS license | Same as Basic + Test Plans |

2. **Security groups** (default):
   | Group | Key Permissions |
   |-------|----------------|
   | Project Administrators | Full project control |
   | Build Administrators | Manage pipelines |
   | Release Administrators | Manage releases |
   | Contributors | Push code, create pipelines, manage work items |
   | Readers | View only |
   | Project Valid Users | Basic access to project |

3. **Configure teams in Azure DevOps**:
   - Project Settings → Teams → New team
   - Each team gets its own:
     - Board and backlog
     - Sprint iterations
     - Dashboard
     - Notifications

4. **Set up area paths and iterations**:
   ```
   Project
   ├── Area: Backend
   │   └── Team: Backend Team
   ├── Area: Frontend
   │   └── Team: Frontend Team
   └── Area: Infrastructure
       └── Team: DevOps Team
   ```

5. **Pipeline-level permissions**:
   - Pipelines → Pipeline → ⋯ → Manage security
   - Control who can: edit, delete, view, queue
   - Resource authorization: approve pipeline access to service connections

### Part C: Compliance Pipeline

```yaml
# Pipeline with security and compliance checks
stages:
  - stage: Compliance
    jobs:
      - job: SecurityChecks
        steps:
          # License compliance
          - script: npx license-checker --onlyAllow "MIT;Apache-2.0;ISC;BSD-2-Clause;BSD-3-Clause"
            displayName: 'Check OSS licenses'

          # Dependency audit
          - script: npm audit --audit-level=high
            displayName: 'Audit dependencies'

          # OWASP dependency check
          - task: dependency-check-build-task@6
            inputs:
              projectName: 'az400-app'
              scanPath: '.'
              format: 'HTML,JUNIT'
            displayName: 'OWASP Dependency Check'
```

## Validation Checklist

- [ ] GitHub organization roles and repository roles memorized
- [ ] Teams configured with repository access
- [ ] Outside collaborator access understood
- [ ] Azure DevOps access levels understood (Stakeholder vs Basic)
- [ ] Security groups and their default permissions known
- [ ] Teams configured with area paths and iterations
- [ ] Pipeline permissions configured
- [ ] License compliance check added to pipeline

## Microsoft Learn References
- [About permissions and security groups](https://learn.microsoft.com/en-us/azure/devops/organizations/security/about-permissions)
- [Access levels in Azure DevOps](https://learn.microsoft.com/en-us/azure/devops/organizations/security/access-levels)
- [GitHub Repository roles](https://docs.github.com/en/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/repository-roles-for-an-organization)
