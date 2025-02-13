# syslog-ng config grammar diff (PR) V2

This action uses [syslog-ng-cfg-helper](https://github.com/syslog-ng/syslog-ng-cfg-helper) to calculate the config differences that a PR introduces and prepares a comment about it, to be picked up by a workflow_run workflow.

For example: https://github.com/alltilla/syslog-ng/pull/123

## Usage

It is recommended to run this action on a `pull_request` trigger. If it is done so, the run acquires the permissions from the originator repo of the PR. If the originator repo is a fork, then it will not be able to comment on an upstream PR. The solution to this is introducing a `workflow_run` workflow, which gets triggered by a workflow, that uses this action, runs in the upstream repo, and has the necessary permissions to comment on a PR.

A `workflow_run` workflow always runs on the source code on master, so it cannot be exploited by changing it in a PR.

The business logic is split to generating the comment in the fork's context done by this action, and commenting on the PR in the upstream's context done by a workflow implemented there. So part of the implementation is done in the repo using this action. If this becomes a hassle in the future, we can create another action (similarly to this one) to be used by the `workflow_run` workflow.


### Comment generating workflow
```yaml
name: Check config grammar changes (PR)

on:
  pull_request:

jobs:
  check-cfg-grammar-changes:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: syslog-ng/syslog-ng-cfg-diff-pr@v2
        continue-on-error: true
```

### Commenting workflow
```yaml
name: Comment config grammar changes (PR)

on:
  workflow_run:
    workflows: [Check config grammar changes (PR)]
    types:
      - completed

permissions:
  pull-requests: write

jobs:
  update-or-remove-comment-on-pr:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
        # The download-artifact action cannot download artifacts from other workflows.
        # Copied from https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_run
      - name: Download comment artifact
        uses: actions/github-script@v6
        with:
          script: |
            let allArtifacts = await github.rest.actions.listWorkflowRunArtifacts({
               owner: context.repo.owner,
               repo: context.repo.repo,
               run_id: context.payload.workflow_run.id,
            });
            let matchArtifact = allArtifacts.data.artifacts.filter((artifact) => {
              return artifact.name == "comment"
            })[0];
            let download = await github.rest.actions.downloadArtifact({
               owner: context.repo.owner,
               repo: context.repo.repo,
               artifact_id: matchArtifact.id,
               archive_format: 'zip',
            });
            let fs = require('fs');
            fs.writeFileSync(`${process.env.GITHUB_WORKSPACE}/comment.zip`, Buffer.from(download.data));

      - name: Unzip comment artifact
        run: unzip comment.zip

      - name: Get PR ID
        id: pr-id
        run: |
          # The pull_requests array always come empty when querying a workflow run's data if it is started from a fork. This might be a GitHub bug.
          # There is also no json field in the `gh run view` command, which could give us the PR ID, so we can only query it based on the fork and branch.
          # See https://github.com/orgs/community/discussions/25220

          PR_ID=$(gh pr view -R "${{ github.repository }}" "${{ github.event.workflow_run.head_repository.owner.login }}:${{ github.event.workflow_run.head_branch }}" --json "number" --jq ".number")
          echo "pr-id=${PR_ID}" >> $GITHUB_OUTPUT

      - name: Update or remove PR comment
        run: |
          EXISTING_COMMENT_ID=$( \
            gh api \
              -H "Accept: application/vnd.github+json" \
              -H "X-GitHub-Api-Version: 2022-11-28" \
              /repos/${{ github.repository }}/issues/${{ steps.pr-id.outputs.pr-id }}/comments \
              --jq '.[] | select(.user.login=="github-actions[bot]") | select(.user.type=="Bot") | select(.user.id==41898282) | select(.body | startswith("#### This Pull Request introduces config grammar changes")) | .id')

          if [[ -s "${GITHUB_WORKSPACE}/comment" ]]; then
            if [[ -n "${EXISTING_COMMENT_ID}" ]]; then
              gh api \
                -H "Accept: application/vnd.github+json" \
                -H "X-GitHub-Api-Version: 2022-11-28" \
                --method PATCH \
                /repos/${{ github.repository }}/issues/comments/${EXISTING_COMMENT_ID} \
                --input ${GITHUB_WORKSPACE}/comment
              echo -e "\n::notice::Updated comment: ${EXISTING_COMMENT_ID}"
            else
              gh api \
                -H "Accept: application/vnd.github+json" \
                -H "X-GitHub-Api-Version: 2022-11-28" \
                --method POST \
                /repos/${{ github.repository }}/issues/${{ steps.pr-id.outputs.pr-id }}/comments \
                --input ${GITHUB_WORKSPACE}/comment
              echo -e "\n::notice::Created new comment"
            fi
          else
            if [[ -n "${EXISTING_COMMENT_ID}" ]]; then
              gh api \
                -H "Accept: application/vnd.github+json" \
                -H "X-GitHub-Api-Version: 2022-11-28" \
                --method DELETE \
                /repos/${{ github.repository }}/issues/comments/${EXISTING_COMMENT_ID}
              echo -e "\n::notice::Removed comment: ${EXISTING_COMMENT_ID}"
            else
              echo -e "\n::notice::Nothing to do"
            fi
          fi
```

## License

The scripts in this project are released under the [MIT License](LICENSE).
