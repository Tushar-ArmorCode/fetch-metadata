<h1 align="center">
    <picture>
        <source media="(prefers-color-scheme: light)" srcset="https://user-images.githubusercontent.com/7659/174594540-5e29e523-396a-465b-9a6e-6cab5b15a568.svg">
        <source media="(prefers-color-scheme: dark)" srcset="https://user-images.githubusercontent.com/7659/174594559-0b3ddaa7-e75b-4f10-9dee-b51431a9fd4c.svg">
        <img src="https://user-images.githubusercontent.com/7659/174594540-5e29e523-396a-465b-9a6e-6cab5b15a568.svg" alt="Dependabot" width="336">
    </picture>
</h1>

# Fetch Metadata Action

**Name:** `dependabot/fetch-metadata`

Extract information about the dependencies being updated by a Dependabot-generated PR.

## Usage instructions

Create a workflow file that contains a step that uses: `dependabot/fetch-metadata@v1`, e.g.

```yaml
-- .github/workflows/dependabot-prs.yml
name: Dependabot Pull Request
on: pull_request_target
jobs:
  build:
    permissions:
      pull-requests: read
    runs-on: ubuntu-latest
    steps:
    - name: Fetch Dependabot metadata
      id: dependabot-metadata
      uses: dependabot/fetch-metadata@v1
      with:
        alert-lookup: true
        compat-lookup: true
        github-token: "${{ secrets.PAT_TOKEN }}"
```

Supported inputs are:

- `github-token` (string)
  - The `GITHUB_TOKEN` secret
  - Defaults to `${{ github.token }}`
  - Note: this must be set to a [personal access token (PAT)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) if you enable `alert-lookup` or `compat-token`.
- `alert-lookup` (boolean)
  - If `true`, then populate the `alert-state`, `ghsa-id` and `cvss` outputs.
  - Defaults to `false`
- `compat-lookup` (boolean)
  - If `true`, then populate the `compatibility-score` output.
  - Defaults to `false`
- `skip-commit-verification` (boolean)
  - If `true`, then the action will not expect the commits to have a verification signature. **It is required to set this to 'true' in GitHub Enterprise Server**
  - Defaults to `false`
- `skip-verification` (boolean)
    - If `true`, the action will not validate the user or the commit verification status
    - Defaults to `false`

Subsequent actions will have access to the following outputs:

- `steps.dependabot-metadata.outputs.dependency-names`
  - A comma-separated list of the package names updated by the PR.
- `steps.dependabot-metadata.outputs.dependency-type`
  - The type of dependency has determined this PR to be.  Possible values are: `direct:production`, `direct:development` and `indirect`.  See [the `allow` documentation](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/configuration-options-for-dependency-updates#allow) for descriptions of each.
- `steps.dependabot-metadata.outputs.update-type`
  - The highest semver change being made by this PR, e.g. `version-update:semver-major`. For all possible values, see [the `ignore` documentation](https://docs.github.com/en/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/configuration-options-for-dependency-updates#ignore).
- `steps.dependabot-metadata.outputs.updated-dependencies-json`
  - A JSON string containing the full information about each updated Dependency.
- `steps.dependabot-metadata.outputs.directory`
  - The `directory` configuration that was used by dependabot for this updated Dependency.
- `steps.dependabot-metadata.outputs.package-ecosystem`
  - The `package-ecosystem` configuration that was used by dependabot for this updated Dependency.
- `steps.dependabot-metadata.outputs.target-branch`
  - The `target-branch` configuration that was used by dependabot for this updated Dependency.
- `steps.dependabot-metadata.outputs.previous-version`
  - The version that this PR updates the dependency from.
- `steps.dependabot-metadata.outputs.new-version`
  - The version that this PR updates the dependency to.
- `steps.dependabot-metadata.outputs.alert-state`
  - If this PR is associated with a security alert and `alert-lookup` is `true`, this contains the current state of that alert (OPEN, FIXED or DISMISSED).
- `steps.dependabot-metadata.outputs.ghsa-id`
  - If this PR is associated with a security alert and `alert-lookup` is `true`, this contains the GHSA-ID of that alert.
- `steps.dependabot-metadata.outputs.cvss`
  - If this PR is associated with a security alert and `alert-lookup` is `true`, this contains the CVSS value of that alert (otherwise it contains 0).
- `steps.dependabot-metadata.outputs.compatibility-score`
  - If this PR has a known compatibility score and `compat-lookup` is `true`, this contains the compatibility score (otherwise it contains 0).
- `steps.dependabot-metadata.outputs.maintainer-changes`
  - Whether or not the the body of this PR contains the phrase "Maintainer changes" which is an indicator of whether or not any maintainers have changed.

**Note:** By default, these outputs will only be populated if the target Pull Request was opened by Dependabot and contains
**only** Dependabot-created commits. To override, see `skip-commit-verification` / `skip-verification`.

This metadata can be used along with Action's [expression syntax](https://docs.github.com/en/actions/reference/context-and-expression-syntax-for-github-actions#functions) and the [GitHub CLI](https://github.com/cli/cli) to create
useful automation for your Dependabot PRs.

### Auto-approving

Since the `dependabot/fetch-metadata` Action will set a failure code if it cannot find any metadata, you can
have a permissive auto-approval on all Dependabot PRs like so:

```yaml
name: Dependabot auto-approve
on: pull_request_target
permissions:
  pull-requests: write
jobs:
  dependabot:
    runs-on: ubuntu-latest
    # Checking the author will prevent your Action run failing on non-Dependabot PRs
    if: ${{ github.event.pull_request.user.login == 'dependabot[bot]' }}
    steps:
      - name: Dependabot metadata
        id: dependabot-metadata
        uses: dependabot/fetch-metadata@v1
      - uses: actions/checkout@v3
      - name: Approve a PR if not already approved
        run: |
          gh pr checkout "$PR_URL" # sets the upstream metadata for `gh pr status`
          if [ "$(gh pr status --json reviewDecision -q .currentBranch.reviewDecision)" != "APPROVED" ];
          then gh pr review --approve "$PR_URL"
          else echo "PR already approved, skipping additional approvals to minimize emails/notification noise.";
          fi
        env:
          PR_URL: ${{github.event.pull_request.html_url}}
          GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
```

### Enabling auto-merge

If you are using [the auto-merge feature](https://docs.github.com/en/github/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/automatically-merging-a-pull-request) on your repository,
you can set up an action that will enable Dependabot PRs to merge once CI and other [branch protection rules](https://docs.github.com/en/github/administering-a-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule) are met.  (Note that you must use a [personal access token (PAT)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) when executing the merge instruction.)

For example, if you want to automatically merge all patch updates to Rails:

```yaml
name: Dependabot auto-merge
on: pull_request_target
permissions:
  pull-requests: write
  contents: write
jobs:
  dependabot:
    runs-on: ubuntu-latest
    if: ${{ github.event.pull_request.user.login == 'dependabot[bot]' }}
    steps:
      - name: Dependabot metadata
        id: dependabot-metadata
        uses: dependabot/fetch-metadata@v1
      - name: Enable auto-merge for Dependabot PRs
        if: ${{contains(steps.dependabot-metadata.outputs.dependency-names, 'rails') && steps.dependabot-metadata.outputs.update-type == 'version-update:semver-patch'}}
        run: gh pr merge --auto --merge "$PR_URL"
        env:
          PR_URL: ${{github.event.pull_request.html_url}}
          GH_TOKEN: ${{secrets.GITHUB_TOKEN}}
```

### Labelling

If you have other automation or triage workflows based on GitHub labels, you can configure an action to assign these based on the metadata.

For example, if you want to flag all production dependency updates with a label:

```yaml
name: Dependabot auto-label
on: pull_request_target
permissions:
  pull-requests: write
  issues: write
  repository-projects: write
jobs:
  dependabot:
    runs-on: ubuntu-latest
    if: ${{ github.event.pull_request.user.login == 'dependabot[bot]' }}
    steps:
      - name: Dependabot metadata
        id: dependabot-metadata
        uses: dependabot/fetch-metadata@v1
      - name: Add a label for all production dependencies
        if: ${{ steps.dependabot-metadata.outputs.dependency-type == 'direct:production' }}
        run: gh pr edit "$PR_URL" --add-label "production"
        env:
          PR_URL: ${{github.event.pull_request.html_url}}
          GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
```

## Notes for project maintainers:

<details><summary>:book: Release guide</summary>
<p>

  - Dependabot PR's:
    - We expect Dependabot PRs to be passing CI and have any changes to the `dist/` folder built for production dependencies
    - Some development dependencies may fail the `dist/` check if they modify the Typescript compilation, these should be updated manually via `npm run build`. See the [`dependabot-build`](https://github.com/dependabot/fetch-metadata/blob/main/.github/workflows/dependabot-build.yml) action for details.
  - Checkout and update `main`, then use the `bin/bump-version` script to create a release PR
      ```bash
      git checkout main
      git pull
      bin/bump-version minor # major | minor | patch
      ```
  - Merge the PR after getting it reviewed
  - Create a new release tagged as `v1.X.X` format:
    - Either via the web UI: https://github.com/dependabot/fetch-metadata/releases/new
    - Or via the CLI:
      ```bash
      gh release create v1.X.X --generate-notes --draft
      > https://github.com/dependabot/fetch-metadata/releases/tag/untagged-XXXXXX
      # Use the generated URL to review/edit the release notes, and then publish it.
      ```
  - Update the `v1` tracking tag to point to the new version
      ```bash
      git fetch --all --tags
      git checkout v1.x.x # Check out the release tag
      git tag -f v1 # Force update the tracking tag
      git push -f --tags
      ```
</p>
</details>
