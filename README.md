# 🛡️ SFDX Assign Permissions

This repository implements a simple GitHub composite action for assigning permission sets, permission set groups and permission set licenses to Salesforce org users from within CI/CD automations. It is the missing step right after a deployment: a freshly created scratch org contains the metadata, but the admin user has none of the permissions the deployed permission sets are supposed to grant.

## Usage

After installing the SF CLI, authorizing the relevant org and deploying the metadata, the permissions can be assigned as follows:

```yaml
# Least-privilege token: the workflow only reads the repository and talks to
# Salesforce via the CLI, it never writes back to GitHub.
permissions:
  contents: read

jobs:
  scratch-org-ci:
    name: Scratch Org CI
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7.0.1
        with:
          persist-credentials: false # the GITHUB_TOKEN is not needed after the checkout - don't leave it in .git/config

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@v1.1.3

      - name: Authorize DevHub
        uses: svierk/sfdx-login@v1.4.3
        with:
          sfdx-url: ${{ secrets.SFDX_AUTH_URL }}
          set-default-dev-hub: true
          alias: devhub

      - name: Create Scratch Org
        id: scratch
        uses: svierk/sfdx-create-scratch-org@v1.3.2
        with:
          target-dev-hub: devhub
          definition-file: config/project-scratch-def.json
          duration-days: 1

      - name: Deploy Metadata
        uses: svierk/sfdx-deploy@v1.2.2
        with:
          source-dir: force-app
          target-org: ${{ steps.scratch.outputs.username }}
          destructive-changes: false # a fresh scratch org has nothing to delete

      - name: Assign Permissions
        uses: svierk/sfdx-assign-permissions@v1.0.0
        with:
          permission-sets: 'DreamHouse,CloudHouse'
          permission-set-groups: 'Sales_Manager'
          target-org: ${{ steps.scratch.outputs.username }}

      # ... run tests, etc. ...

      - name: Delete Scratch Org
        if: always() # release the org even when a previous step failed
        uses: svierk/sfdx-delete-scratch-org@v1.1.3
        with:
          target-org: ${{ steps.scratch.outputs.username }}
```

All four list inputs - `permission-sets`, `permission-set-groups`, `permission-set-licenses` and `on-behalf-of` - accept a comma-separated string as well as a multi-line block, and the two notations can be mixed freely within one step. A comma-separated string keeps short lists on a single line, a block stays readable once a list grows:

```yaml
- name: Assign Permissions
  uses: svierk/sfdx-assign-permissions@v1.0.0
  with:
    permission-set-licenses: 'SalesforceCPQ' # a single value is simply a one-element list
    permission-set-groups: 'Sales_Manager,Service_Agent' # comma-separated
    permission-sets: | # multi-line block
      DreamHouse
      CloudHouse
      mypkg__Package_Admin
    on-behalf-of: 'integration-user@example.com,test-user@example.com'
    target-org: awesome-org
```

Surrounding whitespace and blank lines are ignored, so `'DreamHouse, CloudHouse'` is read exactly like the tightly packed `'DreamHouse,CloudHouse'`. Every permission in the example above is assigned to every user in `on-behalf-of`.

The following actions were also used in the example on this page:

- [SFDX CLI Setup](https://github.com/svierk/sfdx-cli-setup) | Installs the Salesforce CLI and related plugins
- [SFDX Login](https://github.com/svierk/sfdx-login) | Handles Salesforce login using a SFDX auth URL or a JSON web token (JWT)
- [SFDX Create Scratch Org](https://github.com/svierk/sfdx-create-scratch-org) | Creates the scratch org the permissions are assigned in
- [SFDX Deploy](https://github.com/svierk/sfdx-deploy) | Deploys the metadata that contains the permission sets and groups
- [SFDX Delete Scratch Org](https://github.com/svierk/sfdx-delete-scratch-org) | Releases the scratch org again at the end of the run

## Inputs

| Name | Required | Default | Description |
| ---- | -------- | ------- | ----------- |
| `permission-sets` | no | | Permission sets to assign, comma-separated or multi-line. Packaged permission sets can be referenced as `namespace__PermissionSet`. |
| `permission-set-groups` | no | | Permission set groups to assign, comma-separated or multi-line, referenced by their API name (`DeveloperName`). |
| `permission-set-licenses` | no | | Permission set licenses to assign, comma-separated or multi-line, referenced by their API name (`DeveloperName`), e.g. `SalesforceCPQ`. |
| `on-behalf-of` | no | | Org users that receive the permissions, comma-separated or multi-line. Defaults to the user the target org is authorized as - the admin user in a typical scratch org run. |
| `target-org` | no | | Username or alias of the target org. Not required if the default org is set. |
| `api-version` | no | | Override the api version used for api requests, e.g. `59.0`. |
| `wait` | no | `5` | Minutes to wait for permission set groups to finish recalculating before they are assigned. Set to `0` to assign them immediately. |
| `ignore-existing` | no | `true` | Treat "already assigned" as success, so re-runs stay green. Set to `false` to let duplicate assignments fail the step. |
| `step-summary` | no | `true` | Write a result section to the GitHub Actions [job summary](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary). Set to `false` to avoid collisions with a custom workflow summary. |

At least one of `permission-sets`, `permission-set-groups` or `permission-set-licenses` must be provided - an invocation that would assign nothing fails the step instead of silently passing.

## Assignment order

The three types are assigned in a fixed order, because they build on each other:

1. **Permission set licenses** - a permission set can only grant what the user is licensed for, so the licenses are assigned first.
2. **Permission sets** - assigned via `sf org assign permset`.
3. **Permission set groups** - assigned last, after the groups have finished recalculating.

## Permission set groups

The Salesforce CLI has no command for permission set groups. `sf org assign permset` resolves every name against the `PermissionSet` object and fails with _"Permission set … not found in target org"_ when handed the API name of a group. This action therefore resolves groups itself: it looks up the `PermissionSetGroup` record via SOQL and creates the `PermissionSetAssignment` record through `sf data create record`.

Two things follow from that, and both are handled by the action:

- **Recalculation.** A group that was just deployed is recalculated asynchronously and only grants its permissions once its `Status` is `Updated`. Assigning it while it is still `Updating` or `Outdated` can leave the user without the very permissions the pipeline just deployed. The action polls the status for up to `wait` minutes, aborts early with a clear error if the recalculation ends in `Failed`, and - if the group is still not ready when the wait elapses - emits a warning and assigns it anyway rather than failing the pipeline on the timeout alone.
- **User resolution.** Because the assignment record is created directly, the assignee has to be resolved to a `User` record. Values passed to `on-behalf-of` are treated the same way the CLI treats them: a value matching a CLI alias is resolved to its username, anything else is assumed to be a username already. Do not use the `Alias` field of the `User` object. Without `on-behalf-of`, the user the target org is authorized as is used.

## Re-runs and idempotency

Assigning a permission a user already has makes Salesforce return a `duplicate value found` error. The CLI swallows this for permission set licenses, but not for permission sets - which makes any re-run against a long-lived sandbox or dev org fail on the second attempt.

With the default `ignore-existing: true` the action closes that gap: existing permission set group assignments are detected up front and skipped, and duplicate errors reported by `sf org assign permset` are downgraded to a notice. Every other failure - a permission set that does not exist in the org, a missing license, an authorization problem - still fails the step. Set `ignore-existing: false` to get the raw CLI behaviour back.

## Security & versioning

Every `uses:` reference in the snippets above is **pinned to an exact release tag**, e.g. `svierk/sfdx-assign-permissions@v1.0.0`. Do the same in your own pipelines:

- **Never reference a mutable ref** such as `@main` or `@v1`. It runs whatever code sits on that branch/tag at run time - with access to your org credentials - so a compromised or rewritten ref would run unnoticed.
- **Good - pin to an exact release tag** (`@v1.0.0`). Readable, concrete, and bumped through reviewed pull requests.
- **Strictest - pin to a full-length commit SHA** (`@a1b2c3d…`) with the version as a trailing comment. A SHA can never be re-pointed by the publisher; the cost is readability. Worth it for actions from publishers you don't control.
- **Enable [Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot) for `github-actions`** in the repository that hosts the workflow, so those pins are bumped for you instead of silently ageing.

Beyond pinning, a few rules are worth following in the workflow that calls this action:

- **Secrets travel as secrets** - hand the SFDX Auth URL straight into the `sfdx-url` input of [sfdx-login](https://github.com/svierk/sfdx-login). Never interpolate a secret into a `run:` script via `${{ ... }}` - that allows command injection and can leak the value into the log. This action follows the same rule internally: every input is exposed to the shell as an **environment variable** and referenced as `"$PERMISSION_SETS"`, `"$TARGET_ORG"` and so on. Values that end up in a SOQL query are escaped before they are interpolated.
- **Least-privilege `GITHUB_TOKEN`** - declare a `permissions:` block granting only what the job needs. Assigning permissions needs no write access at all, so `contents: read` is enough.
- **`persist-credentials: false` on checkout** - the token is not written to `.git/config`, so later steps (SF CLI, third-party actions) cannot reuse it.
- **Validate pull requests with `pull_request`, never `pull_request_target`** - the latter runs with the base repository's secrets, which would let a fork grant itself permissions in your org.

Resolving the default assignee reads the org details, whose payload contains a live access token. The token is registered as a masked secret before anything is written to the log.

## References

The assignment options supported by this GitHub composite action can be found in the Salesforce CLI Command Reference here:

- [org assign permset](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_org_assign_permset.html)
- [org assign permsetlicense](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_org_assign_permsetlicense.html)
- [data create record](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_data_create_record.html) (used for permission set groups)
- [data query](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_data_query.html)

Background on the objects involved:

- [PermissionSetAssignment](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionsetassignment.htm)
- [PermissionSetGroup](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_permissionsetgroup.htm)

## Releases

Latest release notes can be found on the [release page](https://github.com/svierk/sfdx-assign-permissions/releases).

## License

The scripts and documentation in this project are released under the [MIT License](https://github.com/svierk/sfdx-assign-permissions/blob/main/LICENSE).
