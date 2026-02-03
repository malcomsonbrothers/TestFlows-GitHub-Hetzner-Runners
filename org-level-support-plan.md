# Organization-Level Runner Support Implementation Plan

## Security Warning

> **Self-hosted runners should only be used with PRIVATE repositories.**
>
> From GitHub docs: "Any user can open pull requests against a public repository and compromise the environment. Similarly, be cautious when using self-hosted runners on private or internal repositories, as anyone who can fork the repository and open a pull request (generally those with read access) are able to compromise the self-hosted runner environment."

This is why **Runner Groups** exist - they let you restrict which repositories can access your runners.

---

## Executive Summary

TestFlows-GitHub-Hetzner-Runners currently operates in repository-level mode only, using `GITHUB_REPOSITORY=owner/repo` and calling repo-level GitHub API endpoints. This plan details the changes needed to add organization-level support where a single instance can manage runners for all repositories within a GitHub organization.

**Key Insight from GitHub Documentation**: Organization-level runners are registered with `--url https://github.com/ORG` instead of `--url https://github.com/ORG/REPO`. GitHub automatically handles job routing to org-level runners based on label matching and runner group access policies.

---

## 1. Summary of Changes Needed

### Configuration Layer
1. Add new `--github-organization` CLI flag (mutually exclusive with `--github-repository`)
2. Add `github_organization` config field
3. Update config validation to ensure exactly one of repository/organization is specified
4. Add optional `--runner-group` flag for org-level runner placement

### Scale-Up Service (`scale_up.py`)
1. Abstract job discovery to support both org and repo level
2. Modify registration token retrieval to use org-level endpoint when applicable
3. Update server setup to pass organization URL to startup scripts
4. Handle cross-repo job tracking with org-level runners

### Scale-Down Service (`scale_down.py`)
1. Modify runner listing to use org-level API
2. Update runner removal to use org-level endpoint
3. Preserve existing repo-level logic when in repo mode

### Startup Scripts
1. Modify `startup-x64.sh` and `startup-arm64.sh` to accept either repo or org URL
2. Add new environment variable `GITHUB_ORGANIZATION` or `GITHUB_URL`

### Other Components
1. Update metrics to track org-level data
2. Modify dashboard to display org-level information
3. Update `servers.py`, `estimate.py`, `projects.py` for org support

---

## 2. File-by-File Breakdown

### 2.1 `testflows/github/hetzner/runners/args.py` (187 lines)

**Changes Required**:
- Add `--github-organization` argument type and parser
- Add `--runner-group` argument for specifying runner group (org-level only)

```python
# New addition around line 133
def organization_type(v):
    """GitHub organization name type argument."""
    if v is not None and "/" in v:
        raise ArgumentTypeError(f"organization name should not contain '/': {v}")
    return v
```

### 2.2 `testflows/github/hetzner/runners/config/config.py` (802 lines)

**Changes Required**:

1. Add new fields to `Config` dataclass (around line 113):
```python
github_organization: str = os.getenv("GITHUB_ORGANIZATION")
runner_group: str = "Default"
```

2. Update `check()` method (around line 214) to validate configuration:
```python
def check(self, *parameters):
    """Check mandatory configuration parameters."""
    if not parameters:
        # Either github_repository OR github_organization must be set
        if not self.github_repository and not self.github_organization:
            print("argument error: either --github-repository or --github-organization must be specified", file=sys.stderr)
            sys.exit(1)
        if self.github_repository and self.github_organization:
            print("argument error: --github-repository and --github-organization are mutually exclusive", file=sys.stderr)
            sys.exit(1)
        parameters = ["github_token", "hetzner_token"]
    # ... existing validation ...
```

3. Add helper properties:
```python
@property
def is_org_mode(self) -> bool:
    """Return True if operating in organization mode."""
    return self.github_organization is not None

@property
def github_url(self) -> str:
    """Return the GitHub URL for runner registration."""
    if self.is_org_mode:
        return f"https://github.com/{self.github_organization}"
    return f"https://github.com/{self.github_repository}"

@property
def github_entity(self) -> str:
    """Return org name or repo full name."""
    return self.github_organization or self.github_repository
```

### 2.3 `testflows/github/hetzner/runners/scale_up.py` (1773 lines)

**This is the most critical file requiring changes.**

**Key Changes**:

1. **Registration Token Retrieval** (around line 158-169):

```python
def get_registration_token_url(config):
    """Get the appropriate registration token URL based on mode."""
    if config.is_org_mode:
        return f"https://api.github.com/orgs/{config.github_organization}/actions/runners/registration-token"
    return f"https://api.github.com/repos/{config.github_repository}/actions/runners/registration-token"
```

2. **Server Setup Function** (around line 143-261):

```python
# Pass the GitHub URL and mode to the startup script
github_url = config.github_url
runner_group = config.runner_group

ssh(
    server,
    f"'sudo -u ubuntu "
    f'CACHE_DIR="/mnt/{cache_volume_name}" '
    f'GITHUB_URL="{github_url}" '  # Changed from GITHUB_REPOSITORY
    f'GITHUB_RUNNER_TOKEN="{GITHUB_RUNNER_TOKEN}" '
    f'GITHUB_RUNNER_GROUP="{runner_group}" '  # Now configurable
    f'GITHUB_RUNNER_LABELS="{runner_labels}" '
    # ... rest of script ...
```

3. **Workflow Run Discovery** (around line 1393-1407):

For org mode, we don't need to poll every repo for jobs. We register org-level runners with appropriate labels, and GitHub handles routing. We just need to:
- Monitor runner availability
- Scale up when runners are all busy
- Scale down when runners are idle

4. **Runner Listing** (around line 1456-1464):

```python
def get_self_hosted_runners(github, config):
    """Get self-hosted runners based on mode."""
    if config.is_org_mode:
        org = github.get_organization(config.github_organization)
        return [r for r in org.get_self_hosted_runners()
                if r.name.startswith(runner_name_prefix)]
    else:
        repo = github.get_repo(config.github_repository)
        return [r for r in repo.get_self_hosted_runners()
                if r.name.startswith(runner_name_prefix)]
```

5. **Main Scale-Up Loop** (around line 1362-1368):

```python
with Action("Logging in to GitHub"):
    github = Github(auth=Auth.Token(github_token), per_page=100)

if config.is_org_mode:
    with Action(f"Getting organization {config.github_organization}"):
        org = github.get_organization(config.github_organization)
else:
    with Action(f"Getting repository {config.github_repository}"):
        repo: Repository = github.get_repo(config.github_repository)
```

### 2.4 `testflows/github/hetzner/runners/scale_down.py` (771 lines)

**Changes Required**:

1. **Runner Listing** (around line 326-332):
```python
if config.is_org_mode:
    org = github.get_organization(config.github_organization)
    runners: list[SelfHostedActionsRunner] = org.get_self_hosted_runners()
else:
    runners: list[SelfHostedActionsRunner] = repo.get_self_hosted_runners()
```

2. **Runner Removal** (around line 644):
```python
if config.is_org_mode:
    org.remove_self_hosted_runner(unused_runner.runner)
else:
    repo.remove_self_hosted_runner(unused_runner.runner)
```

### 2.5 Startup Scripts

**`startup-x64.sh`** and **`startup-arm64.sh`**:

```bash
# Support both GITHUB_URL (new) and GITHUB_REPOSITORY (legacy)
if [ -n "${GITHUB_URL}" ]; then
    RUNNER_URL="${GITHUB_URL}"
else
    RUNNER_URL="https://github.com/${GITHUB_REPOSITORY}"
fi

./config.sh --unattended --replace --url "${RUNNER_URL}" --token ${GITHUB_RUNNER_TOKEN} ...
```

### 2.6 `testflows/github/hetzner/runners/bin/github-hetzner-runners`

**Changes Required**:

1. Add new CLI argument:
```python
parser.add_argument(
    "--github-organization",
    metavar="name",
    type=str,
    help="GitHub organization name (mutually exclusive with --github-repository)",
)
```

2. Add runner group argument:
```python
parser.add_argument(
    "--runner-group",
    metavar="name",
    type=str,
    help="Runner group name for organization runners, default: Default",
)
```

3. Add mutual exclusivity validation:
```python
if arguments.github_repository and arguments.github_organization:
    parser.error("--github-repository and --github-organization are mutually exclusive")
```

---

## 3. New Config Options and CLI Flags

| Option | CLI Flag | Environment Variable | Config Key | Description |
|--------|----------|---------------------|------------|-------------|
| GitHub Organization | `--github-organization` | `GITHUB_ORGANIZATION` | `github_organization` | Organization name (mutually exclusive with repository) |
| Runner Group | `--runner-group` | `GITHUB_RUNNER_GROUP` | `runner_group` | Runner group for org runners (default: "Default") |

---

## 4. GitHub API Endpoints

### Repository-Level (Existing)
| Operation | Endpoint |
|-----------|----------|
| List runners | `GET /repos/{owner}/{repo}/actions/runners` |
| Get registration token | `POST /repos/{owner}/{repo}/actions/runners/registration-token` |
| Remove runner | `DELETE /repos/{owner}/{repo}/actions/runners/{runner_id}` |
| List workflow runs | `GET /repos/{owner}/{repo}/actions/runs` |

### Organization-Level (New)
| Operation | Endpoint |
|-----------|----------|
| List runners | `GET /orgs/{org}/actions/runners` |
| Get registration token | `POST /orgs/{org}/actions/runners/registration-token` |
| Remove runner | `DELETE /orgs/{org}/actions/runners/{runner_id}` |
| List runner groups | `GET /orgs/{org}/actions/runner-groups` |

**Note**: There is no direct org-level endpoint for queued jobs across all repos. The recommended approach is to:
1. Register runners at org level
2. Let GitHub handle job routing based on labels
3. Monitor runner state rather than polling all repos for jobs

---

## 5. How Job Routing Works with Org-Level Runners

1. **Runner Registration**: When a runner is registered at the organization level (`--url https://github.com/ORG`), it becomes available to all repositories in the organization by default.

2. **Runner Groups**: Organization owners can use runner groups to control which repositories can access which runners:
   - Default group: All repos can access by default
   - Custom groups: Specific repository access policies

3. **Label Matching**: When a job specifies `runs-on: [self-hosted, label1, label2]`, GitHub finds a runner that:
   - Has all the required labels
   - Is in a runner group accessible to that repository
   - Is idle and available

4. **Job Routing Flow**:
   ```
   Job Queued -> GitHub finds matching runner -> Runner picks up job -> Job executes
   ```

**Implementation Implication**: In org mode, we don't need to poll every repo for jobs. We register org-level runners with appropriate labels, and GitHub handles routing.

---

## 6. Security Hardening (CRITICAL)

### The Risk

When you register a runner at org level, by default it can be used by ANY repo in your org. If any of those repos are:
- **Public**: Anyone on the internet can open a PR and run code on your runner
- **Private but with external collaborators**: Those collaborators can run code
- **Forkable internally**: Anyone who can fork can run code via PR

### Mitigation: Runner Groups

GitHub provides **Runner Groups** to restrict which repos can use which runners:

```
Organization Settings → Actions → Runner groups → Create new group
  ├── Name: "private-only"
  ├── Repository access: "Selected repositories"
  └── Select only your private repos
```

### Implementation: Secure by Default

**We should make TestFlows refuse to set up insecurely.**

1. **Add `--private-repos-only` flag** (default: true)
   - When creating org-level runner, automatically create a runner group
   - Restrict that group to only private repositories
   - Refuse to proceed if `--private-repos-only=false` isn't explicitly set

2. **Add warning on startup**:
   ```
   WARNING: This runner is registered at organization level.
   It will ONLY be accessible to private repositories.
   To allow public repos (DANGEROUS), use --dangerously-allow-public-repos
   ```

3. **Validate runner group on startup**:
   - Check if the runner group allows public repos
   - If yes, refuse to start unless `--dangerously-allow-public-repos` is set

### Additional Security Measures (Already in TestFlows)

1. **Ephemeral runners** - Each job gets a fresh Hetzner server, destroyed after
2. **No persistent compromise** - Unlike traditional self-hosted runners, nothing persists between jobs
3. **Network isolation** - Each server is isolated, no shared state

### Runner Group API Endpoints

| Operation | Endpoint |
|-----------|----------|
| List runner groups | `GET /orgs/{org}/actions/runner-groups` |
| Create runner group | `POST /orgs/{org}/actions/runner-groups` |
| Update runner group | `PATCH /orgs/{org}/actions/runner-groups/{group_id}` |
| Add repo to group | `PUT /orgs/{org}/actions/runner-groups/{group_id}/repositories/{repo_id}` |

### New Config Options for Security

| Option | CLI Flag | Default | Description |
|--------|----------|---------|-------------|
| Private repos only | `--private-repos-only` | `true` | Only allow private repos to use runners |
| Allow public repos | `--dangerously-allow-public-repos` | `false` | Explicitly allow public repos (DANGEROUS) |
| Runner group name | `--runner-group` | `"testflows-private"` | Name of runner group to create/use |
| Auto-create group | `--auto-create-runner-group` | `true` | Create runner group if it doesn't exist |

### Implementation Steps for Security

1. On first run with `--github-organization`:
   - Check if runner group exists
   - If not, create it with private-repos-only policy
   - Register runner in that group

2. On subsequent runs:
   - Verify runner group still has secure policy
   - Warn/fail if policy has been changed to allow public repos

3. Add health check:
   - Periodically verify runner group configuration
   - Alert if security posture degrades

---

## 7. Testing Strategy

### Manual Testing Checklist

- [ ] Deploy with `--github-organization` flag
- [ ] Verify runner appears in org settings
- [ ] Trigger workflow in repo A, verify runner picks it up
- [ ] Trigger workflow in repo B, verify runner picks it up
- [ ] Verify scale-down removes unused org runners
- [ ] Test runner group placement
- [ ] Verify metrics show org-level data

---

## 8. Migration Path for Existing Users

### Backward Compatibility
- Existing configurations with `--github-repository` continue to work unchanged
- No changes required for current repo-level deployments

### Migration Steps

1. **Stop existing service**:
   ```bash
   github-hetzner-runners service stop
   ```

2. **Update configuration** to use `--github-organization` instead of `--github-repository`

3. **Reinstall service**:
   ```bash
   github-hetzner-runners --github-organization your-org-name service install
   ```

4. **Verify org-level runners appear** in GitHub Organization Settings -> Actions -> Runners

---

## 9. Estimated Complexity/Effort

| Component | Complexity | Effort (Hours) |
|-----------|------------|----------------|
| CLI/Args (`args.py`) | Low | 2 |
| Config (`config.py`) | Medium | 4 |
| Scale-Up (`scale_up.py`) | High | 16 |
| Scale-Down (`scale_down.py`) | Medium | 8 |
| Startup Scripts | Low | 2 |
| Servers/List/Delete (`servers.py`) | Medium | 4 |
| Main Entry Point (CLI) | Low | 2 |
| Service (`service.py`) | Low | 2 |
| Testing | High | 16 |
| **Total** | | **~56 hours** |

---

## Critical Files for Implementation

1. `testflows/github/hetzner/runners/scale_up.py` - Core scaling logic, registration token retrieval
2. `testflows/github/hetzner/runners/config/config.py` - Configuration dataclass, validation logic
3. `testflows/github/hetzner/runners/scale_down.py` - Runner listing and removal at org level
4. `testflows/github/hetzner/runners/scripts/startup-x64.sh` - Runner registration script
5. `testflows/github/hetzner/runners/bin/github-hetzner-runners` - CLI entry point
