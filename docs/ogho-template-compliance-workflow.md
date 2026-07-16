# OGHO template compliance workflow

> **Status:** Draft
>
> **Audience:** Oracle GitHub organization owners and repository-governance administrators
>
> **Applies to:** Repositories in Oracle-owned GitHub Enterprise Cloud organizations

## Purpose

The OGHO template compliance workflow identifies pull requests that do not conform to the required Oracle GitHub repository template. It performs the repository-template checks described by the GitHub Compliance Audit Service (GCAS) against the target repository and revision being reviewed.

The intended deployment is a centrally managed GitHub organization branch ruleset. Target repositories do not add or maintain a workflow file. The organization ruleset injects the approved workflow into pull requests and merge groups, which provides consistent enforcement that a repository administrator cannot remove locally.

The workflow:

- identifies template-compliance problems before a pull request is merged;
- provides file-specific GitHub error annotations and a job summary;
- keeps validation code and workflow configuration in one governed repository; and
- allows staged or organization-wide enforcement through repository targeting criteria.

This workflow covers **template checks only**. It does not generate an SBOM, scan dependencies for vulnerabilities, or verify business approvals. Those remain separate GCAS dependency-check capabilities.

## Checks performed

| Area | Requirement |
| --- | --- |
| Repository name | Uses lowercase letters, digits, and single dashes. |
| Default branch | Is named `main`. |
| Required files | `LICENSE.txt`, `README.md`, and `SECURITY.md` exist at the repository root with the exact filename and case. `CONTRIBUTING.md` is optional by default so repositories can inherit Oracle's organization-wide community health file. |
| License format | `LICENSE.txt` contains printable ASCII text and LF line endings. |
| README | Contains a level-one project title and the Installation, Documentation, Examples, Help, Contributing, Security, and License sections. `How to Run` and `Getting Started` are accepted alternatives to `Installation`. |
| Contributing guide | When a local `CONTRIBUTING.md` is present under the default policy, it contains a level-one title; the Opening Issues, Contributing Code, Pull Request Process, and Code of Conduct sections; and a link to the Oracle Contributor Agreement application. |
| Security policy | `SECURITY.md` is byte-for-byte identical to the root policy on the latest `main` revision of `oracle/template-repo`. |

## How centralized enforcement works

The implementation is maintained in `oracle-samples/ogho-compliance`:

- Composite action: `.github/actions/ogho-template-check/action.yml`
- Validator: `.github/actions/ogho-template-check/check.py`
- Default required workflow: `.github/workflows/required-ogho-template-compliance.yml`
- Required workflow without contributing-guide checks: `.github/workflows/required-ogho-template-compliance-contributing-disabled.yml`
- Canonical security policy: root `SECURITY.md` on the `main` branch of `oracle/template-repo`

For each pull request or merge group in a targeted repository, GitHub runs the selected source workflow in the target repository context. The workflow checks out that target repository and invokes the centrally hosted action at an approved full commit SHA.

This arrangement has two important consequences:

- no workflow file is copied into the target repository; and
- changes to the validator or source workflow are reviewed and released once in `oracle-samples/ogho-compliance`.

Do not configure a ruleset workflow that uses a relative action path such as `./.github/actions/ogho-template-check`. A ruleset workflow executes in the target repository workspace, where that relative path does not exist. The supplied workflows use the fully qualified remote action reference.

The action uses Bash, `curl`, and Python's standard library. It downloads the canonical `SECURITY.md` from `https://raw.githubusercontent.com/oracle/template-repo/main/SECURITY.md`; it does not install dependencies or execute application code from the repository being inspected.

## Choose the required workflow

Select exactly one source workflow for each target repository set:

| Source workflow | Policy | Behavior |
| --- | --- | --- |
| `required-ogho-template-compliance.yml` | `contributing-policy: optional` | A repository-local `CONTRIBUTING.md` may be absent. If present, it is validated. Use this for repositories that can inherit the organization-wide community health file. |
| `required-ogho-template-compliance-contributing-disabled.yml` | `contributing-policy: disabled` | All local contributing-guide checks are skipped. Use only for repositories that have been explicitly excluded from those checks. |

Use separate, mutually exclusive rulesets or repository filters for the two policies. If both rulesets target one repository, both workflows run and both results are required.

Both source workflows skip their validation job when `github.repository` is `oracle-samples/ogho-compliance`. This avoids validating the action source repository as though it were an application template. Ruleset-injected runs use the target repository's context and run normally.

## Configure the organization ruleset

The source workflows must be merged to the default branch of `oracle-samples/ogho-compliance` before GitHub can select them for a ruleset.

As an organization owner:

1. Open the Oracle GitHub organization.
2. Select **Settings**.
3. Under **Code, planning, and automation**, select **Repository → Rulesets**.
4. Select **New ruleset → New branch ruleset**.
5. Give the ruleset a clear name, such as `Required OGHO template compliance`.
6. Set the enforcement status to the non-blocking evaluation mode for the initial rollout, when that mode is available for the organization.
7. Under **Target repositories**, choose one of:
   - **All repositories** for full organization coverage;
   - **Only selected repositories** for a fixed pilot or exception group; or
   - **Repositories matching a filter** to target by system or custom properties.
8. Under **Target branches**, add the default branch as the target. Do not target all branches: required workflows run in the pull request and merge queue experience and can otherwise block direct branch creation or updates.
9. Under **Branch protections**:
   - enable **Require a pull request before merging**; and
   - enable **Require workflows to pass before merging**.
10. In the required-workflow rule, select `oracle-samples/ogho-compliance` as the source repository and select one workflow:
    - `.github/workflows/required-ogho-template-compliance.yml`; or
    - `.github/workflows/required-ogho-template-compliance-contributing-disabled.yml`.
11. Grant bypass only to an approved break-glass team or repository-provisioning GitHub App. Do not grant repository administrators a general bypass.
12. Create the ruleset, review results from the pilot scope, remediate failures, and change the enforcement status to **Active** when ready.

Organization rulesets layer with existing repository rulesets and branch-protection rules. A target repository can add stricter rules, but it cannot weaken the organization rule.

## Targeting strategy

For a staged rollout, use selected repositories or an organization custom property. A custom-property model keeps policy selection explicit and supports automatic coverage as repositories are created or reclassified. For example, define a property that distinguishes:

- repositories using the default optional contributing policy;
- repositories approved for disabled contributing checks; and
- repositories temporarily exempt from template enforcement.

Configure mutually exclusive repository filters for the default and disabled rulesets. Keep exemptions narrow, owned, documented, and time-bound. Once the pilot has established a stable failure rate, expand targeting to all applicable repositories.

## Organization prerequisites

Before activation, confirm that:

- GitHub Actions is enabled for every targeted repository.
- The organization Actions policy permits `actions/checkout` and the action under `oracle-samples/ogho-compliance`.
- GitHub-hosted runners are available, or self-hosted runners provide Bash, `curl`, Python 3.10 or later, and a current Actions runner compatible with `actions/checkout@v7`.
- Runners can make outbound HTTPS requests to `raw.githubusercontent.com` to retrieve the canonical security policy.
- The source workflow repository has suitable visibility. A public workflow can target repositories of any visibility; an internal workflow can target internal and private repositories; a private workflow can target private repositories.
- If the source repository is internal or private, its **Settings → Actions → General → Access** setting permits access from repositories in the organization.
- The source workflow does not use `cancel-in-progress` concurrency, which is unsuitable for ruleset-required workflows.

The supplied workflows request only `contents: read` and require no repository or organization secrets.

## Pull request and merge queue behavior

The source workflows declare:

```yaml
on:
  pull_request:
  merge_group:
```

Ruleset workflows support `pull_request`, `pull_request_target`, and `merge_group`. This implementation uses `pull_request` because the validator needs only the pull request contents and a read-only token. It does not use `push` or `workflow_dispatch` because those are not ruleset-enforcement triggers.

The `merge_group` trigger ensures the required check runs for repositories that use a merge queue. GitHub ignores event filters on ruleset workflows and uses the supported events' default activity types.

## New repository creation

A required workflow cannot run while a repository is being initialized and may therefore block repository creation. During rollout, use the ruleset's evaluation mode where available. For active enforcement, grant narrowly scoped bypass permission to the approved repository-provisioning GitHub App or team so it can initialize a repository. The repository becomes subject to the ruleset after initialization.

If the UI offers **Do not require workflow checks on creation**, enable it only when that behavior matches the organization's provisioning policy.

## Recommended rollout

1. Merge the approved action and both source workflows to the default branch of `oracle-samples/ogho-compliance`.
2. Create the default-policy organization ruleset with a small selected-repository set or custom-property filter.
3. Keep enforcement non-blocking during the pilot and review workflow failures and rule insights.
4. Remediate repository content, Actions policy, runner, and network-access problems.
5. Create a separate ruleset for repositories approved to use the disabled contributing policy, using a mutually exclusive target filter.
6. Expand the target filters to all applicable repositories.
7. Change the rulesets to **Active** and monitor failures, bypass use, and exemptions.

No pull request is required in a target repository to install the workflow. Once its repository and default branch match an active organization ruleset, the check is applied centrally.

## Understanding failures

| Failure | Resolution |
| --- | --- |
| Required workflow does not start | Confirm that the source workflow is on the source repository's default branch, Actions is enabled, repository visibility and source access are compatible, and the organization Actions policy permits the referenced actions. |
| Local action cannot be found | Ensure the source workflow uses the fully qualified `oracle-samples/ogho-compliance/.github/actions/ogho-template-check` reference pinned to an approved full commit SHA, not a relative `./.github/actions/...` path. |
| Invalid repository name | Rename the repository using lowercase words separated by dashes. Coordinate redirects and dependent automation before renaming. |
| Default branch is not `main` | Rename the default branch and update branch-protection, build, deployment, and documentation references. |
| Required file is missing | Add the exact root-level filename from `oracle/template-repo`. `CONTRIBUTING.md` is optional under the default workflow and ignored under the disabled workflow. |
| Invalid `LICENSE.txt` format | Convert the file to printable ASCII and LF line endings. Do not replace the approved license text without the appropriate review. |
| README section is missing | Add the missing heading and relevant project content. Installation may instead be titled How to Run or Getting Started. |
| CONTRIBUTING section or OCA link is missing | Restore the required section or the `https://oca.opensource.oracle.com` reference, or confirm that the repository has been assigned to the approved disabled-policy ruleset. |
| `SECURITY.md` differs | Replace it with the root `SECURITY.md` from the latest `main` revision of `oracle/template-repo`. Project-specific security guidance belongs in product documentation, not as a modification to this policy. |

The action reports all detected content failures in one run so repository owners can remediate them together.

## Maintaining the central workflow

Changes to the Oracle repository template require coordinated action maintenance:

1. Update the root template files in `oracle/template-repo`.
2. Update the validator in `oracle-samples/ogho-compliance` when requirements, sections, or accepted aliases change.
3. Run the validator against the template repository and intentional negative fixtures.
4. Merge the reviewed action change and record its full commit SHA.
5. Update both required workflows to reference that approved SHA.
6. Pilot newly introduced requirements with a limited target set or non-blocking enforcement before organization-wide activation.

A root `SECURITY.md` change becomes canonical as soon as it reaches `oracle/template-repo`'s `main` branch and does not require an action-reference update. Action code changes remain pinned until both required workflows are deliberately updated.

Protect the source repository and central workflows with required reviews and a `CODEOWNERS` rule owned by the OGHO administrators.

## References

- GitHub Compliance Audit Service documentation (Oracle internal; available in the OGHO Confluence space)
- [Oracle OGHO compliance repository](https://github.com/oracle-samples/ogho-compliance)
- [Oracle template repository](https://github.com/oracle/template-repo)
- [GitHub: Require workflows to pass before merging](https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets#require-workflows-to-pass-before-merging)
- [GitHub: Create rulesets for repositories in an organization](https://docs.github.com/en/organizations/managing-organization-settings/creating-rulesets-for-repositories-in-your-organization)
- [GitHub: Troubleshoot ruleset workflows](https://docs.github.com/en/enterprise-cloud@latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/troubleshooting-rules#troubleshooting-ruleset-workflows)
- [GitHub: Control Actions and reusable workflow access](https://docs.github.com/en/organizations/managing-organization-settings/disabling-or-limiting-github-actions-for-your-organization)
