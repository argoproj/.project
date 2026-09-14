# Maintainer automation

This repo's `maintainers.yaml` is the authoritative source for Argo maintainer
membership and groups. The **org team sync**
([scripts/cmd/sync-org-teams](../scripts/cmd/sync-org-teams), run from
[.github/workflows/sync-maintainers.yaml](../.github/workflows/sync-maintainers.yaml))
reconciles the GitHub org teams (`argocd-approvers`, `argocd-maintainers`,
`argo-workflows-*`, `argo-events-*`, `argo-rollouts-*`, `helm-approvers`, ...)
to match the **managed** teams in `maintainers.yaml`. It **creates** any team
that does not exist yet and sets each team's description to `Managed by
automation on .project/maintainers.yaml. DO NOT EDIT manually.` so the GitHub UI
signals that membership is automated. Teams marked `managed: false` (e.g.
`emeritus`) are skipped.

The tool lives in the Go module under [scripts/](../scripts) and uses a
[maintainers.yaml loader](../scripts/internal/maintainers/maintainers.go) that
parses the roster with a real YAML parser rather than ad-hoc text scraping.

> The `argoproj/argoproj` `MAINTAINERS.md` file is **not** generated. It is
> maintained by hand and points back to this `maintainers.yaml` as the source
> of truth for membership; update it in the same change that edits the roster.

## Behaviour

- On a **pull request** touching `maintainers.yaml` or `scripts/**`, or on a
  manual **workflow_dispatch**: the team sync runs in **dry-run** (prints the
  add/remove/description plan). No changes are made.
- On **push to `main`** touching `maintainers.yaml`: the sync runs with
  `--apply`, creating any missing teams and reconciling membership/descriptions.

In dry-run a team that does not exist yet is shown as `[CREATE TEAM]`; with
`--apply` it is created (privacy "closed") before its members are added.

## One-time setup

### 1. Create a GitHub App

Create a GitHub App owned by the `argoproj` org with these permissions:

- **Organization > Members**: Read and write (team membership + description sync).
- **Repository > Metadata**: Read (required by GitHub).

Then:

1. **Install** the App on the `argoproj` org.
2. Note the App's numeric **App ID** (App settings → "App ID").
3. Under "Private keys", **Generate a private key** and download the `.pem`.

The workflow does not store a long-lived token. Instead
[`actions/create-github-app-token`](https://github.com/actions/create-github-app-token)
exchanges the App ID + private key for a short-lived **installation access
token** (valid ~1 hour, auto-revoked when the job ends). This is an App
installation token, not a fine-grained PAT, so it is not subject to the CNCF
enterprise PAT-lifetime restriction.

### 2. Configure the App credentials (this repo)

| Name | Kind | Purpose |
|------|------|---------|
| `APP_ID` | Actions **variable** | The GitHub App's numeric App ID (not secret). |
| `APP_PRIVATE_KEY` | Actions **secret** | The full contents of the App's `.pem` private key. |

Set these under the repo's Settings → Secrets and variables → Actions
(`APP_ID` on the *Variables* tab, `APP_PRIVATE_KEY` on the *Secrets* tab). The
workflow mints the token like so:

```yaml
- uses: actions/create-github-app-token@v2
  id: app-token
  with:
    app-id: ${{ vars.APP_ID }}
    private-key: ${{ secrets.APP_PRIVATE_KEY }}
    owner: ${{ github.repository_owner }}
    repositories: ${{ github.event.repository.name }}
    permission-members: write
- name: Reconcile org teams
  env:
    GITHUB_TOKEN: ${{ steps.app-token.outputs.token }}
  ...
```

If `APP_ID` is unset, the token step is skipped and the sync job logs a notice
and exits cleanly, so the automation fails safe.

### 3. Grant teams repo access

The automation **creates** the teams referenced by `maintainers.yaml`, but it
does not grant them repository access. For the subproject `CODEOWNERS` reviews
to work, an org owner must grant each team the appropriate access to its
repositories (one-time, and for any newly created team). Preview what the sync
will create/change with a dry-run:

```bash
cd scripts && GITHUB_TOKEN=... go run ./cmd/sync-org-teams --maintainers-yaml ../maintainers.yaml
```

`[CREATE TEAM]` lines are teams that will be created on the next `--apply` run.

## Safety / operations

- Least privilege: the App only needs the org **Members** permission; it does
  not need access to any repository contents.
- Dry-run first: keep `--apply` off until a dry-run looks correct.
- Key rotation: rotate the App private key periodically and update the
  repo secret.

## Local usage

Run from the `scripts/` directory (the Go module root):

```bash
cd scripts

# Preview org team changes (no writes; add --apply to write):
GITHUB_TOKEN=... go run ./cmd/sync-org-teams --maintainers-yaml ../maintainers.yaml
```
