# pkgs

Vanity URL configuration and deployment for my personal Go module index: [go.treyburn.dev](https://go.treyburn.dev)

This repo uses [vanity](https://github.com/treyburn/vanity), a CLI tool I wrote that generates static HTML pages enabling custom import paths like `go.treyburn.dev/vanity` that redirect to the actual host repositories (like GitHub, Codeberg, Tangled, or your own personal Forgejo).

In this repo, my generated pages are deployed to Cloudflare Pages on every PR merge (unless given a `skip-release` label) and when a registered project publishes a new release.

Browse the full module index at [go.treyburn.dev](https://go.treyburn.dev).

---

## Adding a new project

### 1. Ensure the module path in the new project was set correctly

In the new project's `go.mod`, the module directive must match the vanity domain, ie: `module go.treyburn.dev/my-new-project`

### 2. Register the module

Add an entry to `.vanity.yml` in this repo:
```yaml
modules:
  # append under the previous values in the `modules` field
  # may also optionally configure overrides specific to this new project
  - name: my-new-project # import path: go.treyburn.dev/my-new-project
    repo: https://github.com/treyburn/my-new-project
```

Then create a PR.

> [!NOTE]                                                                                                                                                   
> When creating a PR which modifies the `.vanity.yml` file - a `skip-release` label will automatically be added to the PR.
> This allows you to have a new module pre-registered with the pkgs repo, but only publish to go.treyburn.dev when an official release of the new module is cut.
> 
> If you want to release these changes immediately when the PR is merged - then simply remove the label.

### 3. Create a GitHub PAT for cross-repo dispatch

The new project needs a fine-grained PAT to trigger deploys on this repo.

1. Go to **GitHub > Settings > Developer settings > Personal access tokens > Fine-grained tokens**
2. Click **Generate new token**
3. Configure:
   - **Token name**: `my-new-project-deploy` (best to keep it scoped to a single repo in case of leak)
   - **Repository access**: Select **Only select repositories** > `treyburn/pkgs`
   - **Permissions**: Repository permissions > **Contents** > Read and write
4. Click **Generate token** and copy it
5. In the new project's repo, go to **Settings > Secrets and variables > Actions**
6. Add a repository secret named `GO_PAGES_PAT` with the token value

### 4. Add a trigger-deploy job to the new project's release workflow

Add this job to the project's `.github/workflows/release.yml` (or equivalent):

```yaml
  trigger-deploy:
    needs: [release]  # whatever your final release job is named
    runs-on: ubuntu-slim
    steps:
      - name: Trigger pkgs deploy
        env:
          GH_TOKEN: ${{ secrets.GO_PAGES_PAT }}
        run: |
          gh api repos/treyburn/pkgs/dispatches \
            -f event_type=vanity-release \
            -f "client_payload[version]=${GITHUB_REF_NAME}"
```

This fires a `repository_dispatch` event to this repo after a successful release, which triggers a fresh `vanity generate` and Cloudflare Pages deploy.
