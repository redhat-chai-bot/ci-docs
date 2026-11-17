---
title: "Pull Request Labels and Commands"
description: An index of OpenShift-specific pull request labels and Prow commands, with links to where each one is explained, configured, and implemented.
weight: 3
---

This page is an index of pull request labels and commands that OpenShift CI adds on top of standard Prow. It links to
the documentation, configuration, and source code that define each one. Where a page and the current configuration or
source code disagree, the configuration and source code describe the current behavior. Command examples on this page
are for reference only.

## Four separate questions

A label or command can play different roles on the way to a merge. Keep these questions apart:

- **Who applies a label?** A plugin may add a label in response to a command, or a person may add one with
  `/label <name>` and remove it with `/remove-label <name>`. *Restricted* labels can only be added or removed this way
  by the teams and users listed under `restricted_labels` in the
  [label plugin](https://github.com/kubernetes-sigs/prow/tree/main/pkg/plugins/label) configuration; repository entries
  add to the global ones.
- **Who can run a command?** Each plugin checks its own permissions. [Command help](https://prow.ci.openshift.org/command-help)
  lists commands and who can use them for the plugins that publish help there. Some OpenShift plugins, such as the
  pipeline controller, do not, so use the pages linked below.
- **What blocks a merge?** Tide merges a pull request only when it matches one of the Tide queries for its repository
  and branch (labels that must be present or absent) and its contexts pass. Tide queries are configured in
  `<org>/_prowconfig.yaml` or `<org>/<repo>/_prowconfig.yaml` under
  [`core-services/prow/02_config`](https://github.com/openshift/release/tree/main/core-services/prow/02_config) in
  `openshift/release`. Required contexts come from required Prow jobs, branch protection, and `tide.context_options`
  in [`_config.yaml`][config]. Unless a repository skips unknown contexts, Tide also requires any other reported
  context that is not configured as optional to succeed. None of the labels below is required in every repository or
  on every branch.
- **Which tests run?** Labels and the `ci/pipeline-gate` check generally do not start tests. Exceptions include `lgtm`
  in repositories that use the pipeline controller's LGTM mode, `pipeline-auto`, and Prow's `ok-to-test`. Tests start
  automatically or on request (for example with `/test`, through the pipeline controller, or through the payload and
  multi-PR plugins), and each test reports its own result.

## Labels

| Label | Purpose | How it is applied | Sources |
| --- | --- | --- | --- |
| `verified`, `verified-later` | Records how the change was or will be verified: `verified` alone for pre-merge verification, both labels for post-merge verification. | Normally added and removed by the Jira lifecycle plugin in response to [`/verified`](#openshift-specific-commands). The plugin removes both labels when new commits are pushed. | [Jira Integration][jira-verified], [plugin source][jlp] |
| `backport-risk-assessed` | Indicates that a pull request to a release branch has been evaluated and considered safe to accept. | Restricted label added with `/label`. | No prose page; the label definition in [`_labels.yaml`][labels], global restrictions in [`_plugins.yaml`][plugins], and repository-specific allowed teams and users in `<org>/<repo>/_pluginconfig.yaml`. |
| `staff-eng-approved` | Indicates that a staff engineer has approved a release branch pull request. | Restricted label added with `/label`. | [Branching Phases][branching-final], [`_labels.yaml`][labels], [`_plugins.yaml`][plugins] |
| `acknowledge-critical-fixes-only` | Records that the person who added it acknowledges a critical-fixes-only merge policy. | Added with `/label`; it is listed under `additional_labels`, not `restricted_labels`. It affects merging only while a repository's Tide query requires it. | [Branching Phases][branching-normal], [`_labels.yaml`][labels], [`_plugins.yaml`][plugins] |
| `ci/severity-critical` | Indicates critical CI severity: infrastructure is broken or severely impacted. Defined for the `openshift` and `openshift-eng` organizations; distinct from `priority/ci-critical`, which is defined only for `openshift/release`. | Restricted label added with `/label`. Tide prioritizes pull requests that carry it before others (`tide.priority` in [`_config.yaml`][config]). | No prose page; [`_labels.yaml`][labels], [`_plugins.yaml`][plugins] |
| `jira/valid-bug`, `jira/valid-reference` | Result of the Jira validation of the issue reference in the pull request title. | Added by the Jira lifecycle plugin. | [Jira Integration][jira] |
| `lgtm`, `approved` | Standard Prow review and `OWNERS` approval. | Added by `/lgtm` and `/approve`. The command is `/approve`; `approved` is the label. | [Standard Prow commands](#standard-prow-commands) |

## Commands

### Standard Prow commands

`/lgtm`, `/approve`, `/hold`, `/test`, `/retest`, `/label`, and similar commands come from upstream Prow. Use
[command help](https://prow.ci.openshift.org/command-help) for the commands available in a repository and who can use
them, the upstream [approvers documentation](https://docs.prow.k8s.io/docs/components/plugins/approve/approvers/) for
how `/lgtm` and `/approve` use `OWNERS` files, and [Onboarding a New Component](/how-tos/onboarding-a-new-component/#who-can-approve)
for how OpenShift repositories use them.

### OpenShift-specific commands

| Command | What it does | Sources |
| --- | --- | --- |
| `/verified by <reason>`<br>`/verified later <@username>`<br>`/verified bypass`<br>`/verified remove` | Adds `verified` for pre-merge verification by the named tests or people; adds `verified` and `verified-later` for post-merge verification by the named people; adds `verified` for a change asserted to be non-functional; or removes both labels. The `/verified` commands are restricted to repository collaborators. | [Jira Integration][jira-verified], [plugin source][jlp] |
| `/jira refresh`, `/jira backport`, `/jira cherrypick`, `/cherrypick` | Rerun Jira validation and automate backports. `/cherrypick` is provided by the separate cherrypick plugin. | [Jira Integration][jira-backport] |
| `/override <context>`<br>`/override-sticky <context>`<br>`/override-cancel [<context>]` | Force a status context or check run to green so that it no longer blocks merging, keep that override when the base branch moves, or remove overrides. An override does not rerun or fix the test. Available to repository administrators, approvers in the top-level `OWNERS` file, and the GitHub teams listed for the repository's organization under `override` in [`_plugins.yaml`][plugins]. | [Overriding Failing CI Jobs][override] (read [the checks before applying an override][override-checks] first), [upstream plugin](https://github.com/kubernetes-sigs/prow/tree/main/pkg/plugins/override) |
| `/payload`, `/payload-job`, `/payload-aggregate`, their `-with-prs` variants, and `/payload-abort` | Build a release payload that includes the pull request and run release-qualification periodic jobs against it, or abort those runs. Some variants start many expensive jobs. | [Payload Commands][payload], [plugin source](https://github.com/openshift/ci-tools/tree/main/cmd/payload-testing-prow-plugin) |
| `/testwith` | Run a test against this pull request combined with other pull requests. The resulting check run is never required for merging. | [Multi-PR Presubmit Testing](/how-tos/multi-pr-presubmit-testing/#testwith-command), [plugin source](https://github.com/openshift/ci-tools/tree/main/cmd/multi-pr-prow-plugin) |
| `/pipeline help`, `/pipeline required`, `/pipeline remaining`, `/pipeline auto`, `/pipeline skip-agentic-mode`, `/pipeline agentic-mode` | List commands, or request and control second-stage dispatch in repositories enrolled in the pipeline controller. Behavior differs in agentic mode. `/pipeline auto` (LGTM mode only) adds the `pipeline-auto` label, and `/pipeline skip-agentic-mode` adds `pipeline-skip-agentic-mode`. | [Creating a Pipeline][pipeline-commands], [controller user guide][pipeline-guide] |
| `/pipeline mark-pipeline-gate [<HEAD>]` | Marks `ci/pipeline-gate` successful for the current HEAD. It does not start tests. | [Below](#the-pipeline-gate) |

## The pipeline gate

[Creating a Pipeline][pipeline-gate] explains the gate and its recovery. In short:

- **What the gate records.** For repositories and target branches enrolled in the pipeline controller, the controller
  publishes a `ci/pipeline-gate` check run for the pull request HEAD. It records that the applicable second-stage tests
  were *dispatched*. It does not wait for or report their results; each test reports its own context.
- **When it blocks merging.** A missing gate blocks merging only on branches where the `ci/pipeline-gate` context is
  required. Once reported, a pending or failed gate blocks Tide like any other context that is not configured as
  optional, unless the repository skips unknown contexts.
- **What `/pipeline mark-pipeline-gate` does.** It completes the controller's gate as successful for the current HEAD,
  or creates a successful gate if one is missing. If a full 40-character commit SHA is given, the command is rejected
  unless that SHA is the current HEAD. Post the command alone in a new comment, with no SHA or with the full SHA. The
  command adds no label, starts no tests, and does not change the context of any test. On agentic-enrolled branches,
  marking the gate also records dispatch for that HEAD as complete, so the controller does not automatically dispatch
  tests for that HEAD.
- **Who can use it.** Members of the repository's GitHub organization and collaborators on the repository. The
  repository and target branch must be enrolled, and the controller must use GitHub App authentication. In
  [agentic mode][pipeline-agentic], the configured Chai account can only complete an existing readiness gate.
- **When to use it.** For recovery, after confirming that the applicable second-stage tests were dispatched or that
  none apply. See [Recover a missing or stuck gate][pipeline-recover].

Enrollment and gate requirements are repository-specific. For example, at the time of writing:

| Repository | Pipeline controller enrollment | `ci/pipeline-gate` required by Tide |
| --- | --- | --- |
| `openshift/origin` | [`config.yaml`][pipeline-config]: listed branches, automatic trigger with agentic mode | On the enrolled branches |
| `openshift/api` | [`lgtm-config.yaml`][pipeline-lgtm-config]: all branches, LGTM trigger with agentic mode | On all branches |

The configuration files are authoritative; check them for the current state. Sources for the pipeline controller:

- Documentation: [Creating a Pipeline][pipeline-gate] and the controller's [user guide][pipeline-guide].
- Implementation: [`cmd/pipeline-controller`](https://github.com/openshift/ci-tools-standalone/tree/main/cmd/pipeline-controller)
  in `openshift/ci-tools-standalone`, which replaced the earlier copy in `openshift/ci-tools`. Command parsing and the
  permission check are in `commands.go`, comment handling in `main.go` and `agentic_events.go`, and gate updates in
  `dispatch_check.go` and `agentic.go`.
- Configuration: enrollment in [`config.yaml`][pipeline-config] and [`lgtm-config.yaml`][pipeline-lgtm-config];
  required contexts under `tide.context_options` in [`_config.yaml`][config].

[config]: https://github.com/openshift/release/blob/main/core-services/prow/02_config/_config.yaml
[labels]: https://github.com/openshift/release/blob/main/core-services/prow/02_config/_labels.yaml
[plugins]: https://github.com/openshift/release/blob/main/core-services/prow/02_config/_plugins.yaml
[jira]: /architecture/jira/
[jira-verified]: /architecture/jira/#the-verified-label
[jira-backport]: /architecture/jira/#cherrypickingbackporting
[jlp]: https://github.com/openshift-eng/jira-lifecycle-plugin
[branching-normal]: /architecture/branching/#normal-development-for-4x-release
[branching-final]: /architecture/branching/#final-stabilization-for-soon-to-be-released-minor-versions
[override]: /how-tos/overriding-failing-ci-jobs/
[override-checks]: /how-tos/overriding-failing-ci-jobs/#things-to-do-before-applying-an-override
[payload]: /release-oversight/pull-request-testing/#payload-commands
[pipeline-commands]: /how-tos/creating-a-pipeline/#run-tests-with-commands
[pipeline-gate]: /how-tos/creating-a-pipeline/#require-the-dispatch-gate
[pipeline-recover]: /how-tos/creating-a-pipeline/#recover-a-missing-or-stuck-gate
[pipeline-agentic]: /how-tos/creating-a-pipeline/#agentic-mode-with-chai--closed-beta
[pipeline-guide]: https://github.com/openshift/ci-tools-standalone/blob/main/cmd/pipeline-controller/documentation.md
[pipeline-config]: https://github.com/openshift/release/blob/main/core-services/pipeline-controller/config.yaml
[pipeline-lgtm-config]: https://github.com/openshift/release/blob/main/core-services/pipeline-controller/lgtm-config.yaml
