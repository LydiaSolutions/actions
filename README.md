# actions

Shared GitHub Actions for configuring SSH access to private GitHub repositories and removing the generated credentials afterward. This repository contains two composite actions; repository checkout or other Git operations remain the responsibility of the calling workflow.

## Actions

- [`private_repositories`](private_repositories/action.yml) accepts a required `ssh-key` input containing a Base64-encoded private SSH key. It decodes the key into a temporary file, sets its permissions to `600`, and writes SSH settings for `github.com` to `~/.ssh/config.d/customconfig`. Its declared `keyfile` output identifies the temporary key file.
- [`clean_ssh`](clean_ssh/action.yml) accepts the required `keyfile` path and deletes it. It also removes **all entries matching `~/.ssh/config.d/*`**, including configuration created by other steps.

## Using the actions

The runner needs Bash, OpenSSH, and the utilities used by the action scripts, including `base64` with support for `-d`. The supplied key must have access to the target repositories. Store its Base64-encoded value in a GitHub Actions secret; Base64 encoding does not encrypt the key.

The runner's SSH configuration must already load `~/.ssh/config.d/*`. The setup action creates a configuration fragment but does not configure SSH to include it.

The following steps illustrate the input and output wiring, pinned to this repository snapshot:

```yaml
steps:
  - name: Configure private repository access
    id: private_repositories
    uses: LydiaSolutions/actions/private_repositories@9505ddbc22074b39569b6f0af749d248d2238e82
    with:
      ssh-key: ${{ secrets.PRIVATE_REPOSITORIES_SSH_KEY }}

  # Add the steps that access private repositories here.

  - name: Remove SSH credentials and configuration
    if: ${{ always() && steps.private_repositories.outputs.keyfile != '' }}
    uses: LydiaSolutions/actions/clean_ssh@9505ddbc22074b39569b6f0af749d248d2238e82
    with:
      keyfile: ${{ steps.private_repositories.outputs.keyfile }}
```

`PRIVATE_REPOSITORIES_SSH_KEY` is an example secret name to configure in the calling repository.

## Operational constraints

The setup action uses the legacy `::set-output` command. Confirm runner support before relying on the `keyfile` output and cleanup wiring above. If the output is unavailable, the guarded cleanup step is skipped and the temporary key can remain on the runner.

The generated SSH configuration disables host key verification for `github.com`. Setup overwrites `customconfig`, and cleanup does not restore previous configuration. Use a runner where these changes and the broad cleanup scope are acceptable.

No automated tests or validation workflow are included in this snapshot.
