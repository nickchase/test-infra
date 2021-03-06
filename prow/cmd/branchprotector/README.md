# Branch Protection Documentation

branchprotector configures [github branch protection] according to a specified
policy.

## Policy configuration

Extend the primary prow [`config.yaml`] document to include a top-level
`branch-protection` key that looks like the following:

```yaml

branch-protection:
  orgs:
    kubernetes:
      repos:
        test-infra:
          # Protect all branches in kubernetes/test-infra
          protect-by-default: true
          # Always allow the org's oncall-team to push
          pushers: ["oncall-team"]
          # Ensure that the extra-process-followed github status context passes.
          # In addition, adds any required prow jobs (aka always_run: true)
          require-contexts: ["extra-process-followed"]

presubmits:
  kubernetes/test-infra:
  - name: fancy-job-name
    context: fancy-job-name
    always_run: true
    spec:  # podspec that runs job
```

This config will:
  * Enable protection for every branch in the `kubernetes/test-infra`
repo.
  * Require `extra-process-followed` and `fancy-job-name` [status contexts] to pass
    before allowing a merge
    - Although it will always allow `oncall-team` to merge, even if required
      contexts fail.
    - Note that `fancy-job-name` is pulled in automatically from the
      `presubmits` config for the repo, if one exists.

### Updating

* Send PR with `config.yaml` changes
* Merge PR
* Done!

Make changes to the policy by modifying [`config.yaml`] in your favorite text
editor and then send out a PR. When the PR merges prow pushes the updated config
. The branchprotector applies the new policies the next time it runs (within
24hrs).

### Advanced configuration

See [`branch_protection.go`] for a complete list of fields allowed
inside `branch-protection`.  It is possible to define a policy at the
`branch-protection`, `org`, `repo` or `branch` level. For example:

```yaml
branch-protection:
  # Protect unless overridden
  protect-by-default: true
  # If protected, always require the cla status context
  require-contexts: ["cla"]
  orgs:
    unprotected-org:
      # Disable protection unless overridden (overrides parent setting of true)
      protect-by-default: false
      repos:
        protected-repo:
          # Inherit protect-by-default config from parent
          # If protected, always require the tested status context
          require-contexts: ["tested"]
          branches:
            secure:
              # Protect the secure branch (overrides inhereted parent setting of false)
              protect-by-default: true
              # Require the foo status context
              require-contexts: ["foo"]
    different-org:
      # Inherits protect-by-default: true setting from above
```

Any `pushers` and `require-contexts` defined at the parent level are appended to
child levels. The `protect-by-default` setting inherits from the parent if
`null` (default) otherwise a child setting overrides the parent setting.

So in the example above:
  * The `secure` branch `unprotected-org/protected-repo`
    - enables protection (set a branch level)
    - requires `foo` `tested` `cla` [status contexts]
      (the latter two are appended by ancestors)
  * All other branches in `unprotected-org/protected-repo`
    - disable protection (inherited from org level)
  * All branches in all other repos in `unprotected-org`
    - disable protection (set at org level)
  * All branches in all repos in `different-org`
    - Enable protection (inherited from branch-protection level)
    - Require the `cla` context to be green to merge (appended by parent)

## Developer docs

Use [`planter.sh`] if [`bazel`] is not already installed on the machine.

### Run unit tests

`bazel test //prow/cmd/branchprotection:all`

### Run locally

`bazel run //prow/cmd/branchprotection -- --help`, which will tell you about the
current flags.

Do a dry run (which will not make any changes to github) with
something like the following command:

```sh
bazel run //prow/cmd/branchprotection -- \
  --config-path=/path/to/config.yaml \
  --github-token-path=/path/to/my-github-token
```

This will say how the binary will actually change github if you add a
`--confirm` flag.

### Deploy local changes to dev cluster

Run things like the following:
```sh
# Build and push image, create job
bazel run //prow/cmd/branchprotection:oneshot.create
# Delete finished job
bazel run //prow/cmd/branchprotection:oneshot.delete
# Describe current state of job
bazel run //prow/cmd/branchprotection:oneshot.describe
```

This will build an image with your local changes, push it to
`STABLE_DOCKER_REPO` and then deploy to `STABLE_PROW_CLUSTER`.

See [`print-workspace-status.sh`] for the definition of these variables.
See [`oneshot-job.yaml`] for details on the deployed job resource.

### Deploy cronjob to production

Follow the standard prow deployment process:

```sh
# build and push image, update tag used in production
prow/bump.sh branchprotector
# apply changes to production
bazel run //prow/cluster:branchprotector.apply
```

See [`prow/bump.sh`] for details on this script.
See [`prow/cluster/branchprotector_cronjob.yaml`] for details on the deployed
job resource.


[`bazel`]: https://bazel.build
[`branch_protection.go`]: branch_protection.go
[`config.yaml`]: /prow/config.yaml
[github branch protection]: https://help.github.com/articles/about-protected-branches/
[`oneshot-job.yaml`]: oneshot-job.yaml
[`planter.sh`]: /planter
[`print-workspace-status.sh`]: ../../../hack/print-workspace-status.sh
[`prow/bump.sh`]: /prow/bump.sh
[`prow/cluster/branchprotector_cronjob.yaml`]: /prow/cluster/branchprotector_cronjob.yaml
[status contexts]: https://developer.github.com/v3/repos/statuses/#create-a-status
