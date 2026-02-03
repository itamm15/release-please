## Release PR GitHub Action

This repository provides a composite GitHub Action that **creates or updates a release pull request** from a head branch (for example `develop`) into a base branch (for example `main`) and **generates a changelog from merge commits**.

The action:

- **Generates a changelog** by listing merge commit subjects between the base and head branches.
- **Finds an existing open PR** from head → base and updates its title and body, or
- **Creates a new PR** if one does not exist.

### Inputs

- **`base`** (required)  
  - **Description**: Base branch to receive the release PR.  
  - **Default**: `main`

- **`head`** (required)  
  - **Description**: Head branch to merge from.  
  - **Default**: `develop`

- **`title`** (optional)  
  - **Description**: Title of the release PR.  
  - **Default**: `Release PR`

- **`token`** (required)  
  - **Description**: GitHub token used by the `gh` CLI to list, create, and update pull requests.  
  - **Typical value**: `${{ secrets.GITHUB_TOKEN }}` or a personal access token (PAT) with `repo` scope.

### How it works

Internally, the action:

1. Checks out the repository with full history (`fetch-depth: 0`).
2. Fetches both the base and head branches from `origin`.
3. Computes the range `origin/<base>..origin/<head>` and writes merge commit subjects to `body.md`.
4. Uses `gh pr list` to detect an existing open PR from `head` into `base`.
5. **If a PR exists**, it updates the PR title and body with `gh pr edit`.  
   **If no PR exists**, it creates a new PR with `gh pr create`.

### Usage

Add a workflow like this in your repository (for example: `.github/workflows/release-pr.yml`):

```yaml
name: Release PR

on:
  workflow_dispatch:
    inputs:
      base:
        description: "Base branch"
        required: true
        default: main
      head:
        description: "Head branch"
        required: true
        default: develop

jobs:
  release-pr:
    runs-on: ubuntu-latest
    steps:
      - name: Create or update release PR
        # Pin to a specific commit for reproducible runs
        uses: itamm15/release-please-action@<commit-sha>
        with:
          base: ${{ github.event.inputs.base }}
          head: ${{ github.event.inputs.head }}
          title: "Release PR: ${{ github.event.inputs.head }} → ${{ github.event.inputs.base }}"
          token: ${{ secrets.GITHUB_TOKEN }}
```

You can also hard-code the branches if you always release from a specific head to base:

```yaml
      - name: Create or update release PR
        # Pin to a specific commit for reproducible runs
        uses: itamm15/release-please-action@<commit-sha>
        with:
          base: main
          head: develop
          token: ${{ secrets.GITHUB_TOKEN }}
```

> **Recommendation**: Always **pin this action by commit SHA**  
> (for example `itamm15/release-please-action@a1b2c3d4e5f6...`) rather than a branch or tag.  
> This ensures your workflows stay reproducible and are not affected by future changes to this repository.

### Example repository

For a concrete example of how this action can be wired into a workflow, see the companion repository:  
[release-please-use-case](https://github.com/itamm15/release-please-use-case).

### Requirements & notes

- **Environment**: Designed for GitHub-hosted runners (e.g. `ubuntu-latest`) where the **`gh` CLI is pre-installed**. If you run on a custom runner, ensure `gh` is installed and authenticated.
- **Permissions**: The token you provide must be allowed to read and write pull requests. The default `GITHUB_TOKEN` usually has sufficient permissions in most repositories.
- **History**: The action itself performs a checkout with `fetch-depth: 0`, so you do **not** need to handle that in your workflow.

### Troubleshooting

- **No PR is created**  
  - Confirm that both `base` and `head` branches exist on `origin`.  
  - Ensure the token has permission to create and edit PRs.

- **Empty or unexpected changelog**  
  - The body is built from **merge commits only** in the range `origin/<base>..origin/<head>`.  
  - If you use fast-forward merges or squash merges without merge commits, the generated list may be empty.

