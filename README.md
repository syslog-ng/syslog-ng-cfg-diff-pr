# syslog-ng config grammar diff (PR) V1

This action uses [syslog-ng-cfg-helper](https://github.com/alltilla/syslog-ng-cfg-helper) to calculate the config differences that a syslog-ng PR introduces then creates/updates a comment about it.

For example: https://github.com/alltilla/syslog-ng/pull/123

## Usage

```yaml
name: Check config grammar changes (PR)

permissions:
  pull-requests: write

on:
  pull_request:

jobs:
  check-cfg-grammar-changes:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: alltilla/syslog-ng-cfg-diff-pr@v1
        continue-on-error: true
```

## License

The scripts in this project are released under the [MIT License](LICENSE).
