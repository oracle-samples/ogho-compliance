# OGHO template compliance workflow

This repository hosts the GitHub Action and ruleset workflows that check Oracle repositories for compliance with the repository requirements defined by [`oracle/template-repo`](https://github.com/oracle/template-repo).

The intended deployment is organization-wide enforcement through a GitHub organization branch ruleset. Target repositories do not add or maintain their own workflow file. The organization ruleset runs a centrally managed workflow against each targeted pull request and merge group, so repository administrators cannot remove the check locally.

## Organization-required workflows

An organization owner selects one of these source workflows in the ruleset's **Require workflows to pass before merging** rule:

| Source workflow | Contributing policy | Use for |
| --- | --- | --- |
| `.github/workflows/required-ogho-template-compliance.yml` | `optional` | Repositories that may inherit Oracle's organization-wide `CONTRIBUTING.md`. An absent local file passes; a local file is validated when present. |
| `.github/workflows/required-ogho-template-compliance-contributing-disabled.yml` | `disabled` | Repositories that are explicitly excluded from contributing-guide validation. |

Target each repository with only one workflow. If both rulesets apply, both checks run.

Both workflows check out the target repository and call the action in this repository at an approved full commit SHA. They skip their validation job only when triggered in this source repository; ruleset-injected runs use the target repository context and run normally.

## Configure the organization ruleset

In the Oracle organization's **Settings**, select **Repository → Rulesets**, create a branch ruleset, and:

1. Target all applicable repositories, selected repositories, or repositories matching a custom-property filter.
2. Target the default branch only.
3. Enable **Require a pull request before merging**.
4. Enable **Require workflows to pass before merging**.
5. Select `oracle-samples/ogho-compliance` as the source repository and select exactly one required workflow from the table above.
6. Restrict bypass access to an approved break-glass team or provisioning GitHub App.
7. Evaluate the ruleset on a pilot repository set, remediate failures, then activate it for the intended scope.

The selected workflow must already exist on this repository's default branch. GitHub Actions must be enabled in target repositories, and the organization Actions policy must permit `actions/checkout` and this action.

For prerequisites, rollout guidance, troubleshooting, and maintenance, see the [OGHO template compliance workflow guide](docs/ogho-template-compliance-workflow.md).

## What the check validates

The action checks the repository name and default branch, required root files, license format, required README and contributing-guide sections, and the canonical security policy. It reports failures as GitHub annotations and in the job summary.

It does not maintain copies of the repository templates. The canonical `SECURITY.md` is read from the latest `main` revision of `oracle/template-repo` at runtime.

## License

Copyright (c) 2026 Oracle and/or its affiliates.

Released under the Universal Permissive License v1.0 as shown in [LICENSE.txt](LICENSE.txt).
