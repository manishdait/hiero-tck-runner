# hiero-tck-runner

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A GitHub Action to run the **Hiero SDK Technology Compatibility Kit (TCK)** test suite against a target JSON-RPC server and network node. 

> [!WARNING]
> This is an unofficial GitHub Action designed to simplify SDK testing against [`hiero-ledger/hiero-sdk-tck`](https://github.com/hiero-ledger/hiero-sdk-tck).



## Quick Start

Add this step to your GitHub Actions workflow file (e.g., `.github/workflows/tck-tests.yml`) to test SDK's JSON-RPC server:

```yaml
- name: Run Hiero TCK Test Suite
  uses: manishdait/hiero-tck-runner@main
  with:
    dockerfilePath: './tck/Dockerfile'
    serverEnv: |
      TCK_PORT=8544
```

##  Inputs

### Server under test

| Input | Description | Default |
| ----- | ----------- | ------- |
| `startServer` | Build and run the server from `dockerfilePath`. Set `false` if your workflow starts it. | `true` |
| `dockerfilePath` | Path to the Dockerfile, relative to repository root | `./Dockerfile` |
| `rpcServerPort` | Port the JSON-RPC server listens on | `8544` |
| `serverEnv` | Environment passed to the container, one `KEY=VALUE` per line | `""` |
| `serverStartupTimeout` | Seconds to wait for the server to answer | `120` |

### Network under test

Defaults match [`hiero-solo-action`](https://github.com/hiero-ledger/hiero-solo-action) with `installMirrorNode: true`.

| Input | Description | Default |
| ----- | ----------- | ------- |
| `nodeIp` | Consensus node address | `127.0.0.1:35211` |
| `nodeAccountId` | Consensus node account | `0.0.3` |
| `operatorAccountId` | Operator account | `0.0.2` |
| `operatorPrivateKey` | Operator key (masked in logs) | Solo genesis key |
| `mirrornodeGrpcUrl` | Mirror node gRPC | `127.0.0.1:5600` |
| `mirrornodeRestUrl` | Mirror node REST | `http://127.0.0.1:38081` |
| `mirrornodeRestJavaUrl` | Mirror node Java REST | `http://127.0.0.1:8084` |
| `nodeTimeout` | Consensus request timeout (ms) | `30000` |

### Which tests to run

| Input | Description | Default |
| ----- | ----------- | ------- |
| `tckTag` | Tag, branch or SHA of `hiero-sdk-tck` | `v0.12.0` |
| `testSpec` | Space-separated spec files/globs. Runs **only** these. Overrides `testScript`. | `""` |
| `testGrep` | Only tests whose full title matches this regex | `""` |
| `testScript` | npm script when `testSpec` is empty: `test` (parallel) or `test:serial` | `test:serial` |

### Reporting

| Input | Description | Default |
| ----- | ----------- | ------- |
| `uploadReport` | Upload the mochawesome report as an artifact | `true` |
| `artifactName` | Artifact name. Vary per matrix leg. | `tck-report` |


##  Running a targeted subset

The full suite takes roughly **33 minutes**, almost all of it waiting on the network:
`beforeEach` hooks create accounts and mint tokens on chain. Failures are cheap; setup is not.
So the way to make a PR fast is to run fewer tests, not to make tests faster.

**While implementing one method**, point `testSpec` at its spec file. Only that file is
loaded and compiled, which is much faster than filtering the whole suite with `testGrep`:

```yml
- uses: manishdait/hiero-tck-runner@main
  with:
    testSpec: "src/tests/crypto-service/test-account-create-transaction.ts"
```

Narrow further to a single test with `testGrep`:

```yml
    testSpec: "src/tests/crypto-service/test-account-create-transaction.ts"
    testGrep: "Creates an account with"
```

**For the full suite**, `testScript: test` runs mocha with `--parallel --jobs 7`, which is what
every upstream `hiero-sdk-tck` compatibility workflow uses. The work is network-bound, so this
parallelises well. `test:serial` remains the default.

```yml
    testScript: "test"
```

> [!TIP]
> A common pattern is `testSpec` on pull requests for fast feedback, and the full suite
> nightly on a schedule.


##  Outputs

The action always lets the TCK suite run to completion, then publishes its results before failing the
job. Results come from the mochawesome report the suite writes to `hiero-tck/mochawesome-report/`.

| Output | Description |
| ------ | ----------- |
| `total` | Total number of TCK tests executed |
| `passed` | Number of passing tests |
| `failed` | Number of failing tests |
| `pending` | Number of pending tests |
| `hookFailures` | Failed suite hooks, counted separately from test failures |
| `skipped` | Registered tests that never ran, usually after a hook failure |
| `registered` | Tests registered by the suite, including those that never ran |
| `reportPath` | Path to the mochawesome report directory, empty if no report was produced |

In addition, the action writes a pass/fail table (and a collapsed list of failing tests) to the
[job summary](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary),
uploads the HTML and JSON report as an artifact, and dumps the RPC server container logs when the
suite fails.

To act on the results yourself, give the step an `id` and read its outputs:

```yml
- name: Run TCK test
  id: tck
  uses: manishdait/hiero-tck-runner@main

- name: Report
  if: always()
  run: echo "${{ steps.tck.outputs.passed }}/${{ steps.tck.outputs.total }} TCK tests passed"
```

> [!NOTE]
> `artifactName` must be unique per job. When running this action in a matrix, vary it
> (e.g. `artifactName: tck-report-${{ matrix.sdk }}`) or the artifact upload will fail on
> duplicate names.


##  Requirements and caveats

- **Linux runners only.** The server container is started with `--network host` so it can reach
  Solo on `localhost`. Host networking is a no-op on macOS and Windows runners.
- **Docker and `jq`** must be present. Both are preinstalled on `ubuntu-latest`.
- **The action runs `actions/setup-node`**, which changes the Node version for the rest of the
  job. If your workflow depends on a specific Node version afterwards, re-run `setup-node`.
- **`operatorPrivateKey` is masked** via `::add-mask::`, but action inputs are not secrets.
  The default is the well-known Solo genesis key. Never pass a key with real value.

> [!IMPORTANT]
> `rpcServerPort` tells the action where to *probe*; it does not tell your server where to
> *listen*. Use `serverEnv` to pass the port through in whatever form your server expects
> (`TCK_PORT`, `PORT`, ...), or the two will disagree.

## Usage

A pull-request check that runs only the method being worked on, plus the full suite nightly:

```yml
name: TCK

on:
  pull_request:
  schedule:
    - cron: "0 3 * * *"

permissions:
  contents: read

jobs:
  tck:
    name: TCK
    runs-on: ubuntu-latest
    timeout-minutes: 90

    steps:
      - name: Checkout repository
        uses: actions/checkout@v7.0.1

      - name: Prepare Hiero Solo
        id: solo
        uses: hiero-ledger/hiero-solo-action@v0.24.0
        with:
          installMirrorNode: true

      - name: Run TCK
        id: tck
        uses: manishdait/hiero-tck-runner@main
        with:
          dockerfilePath: './tck/Dockerfile'
          serverEnv: |
            TCK_PORT=8544
          # Fast, targeted run on PRs; whole suite in parallel on the nightly.
          testSpec: ${{ github.event_name == 'pull_request' && 'src/tests/crypto-service/test-account-create-transaction.ts' || '' }}
          testScript: "test"
          artifactName: tck-report-${{ github.event_name }}

      - name: Summarise
        if: always()
        run: |
          echo "${{ steps.tck.outputs.passed }}/${{ steps.tck.outputs.total }} passed, \
                ${{ steps.tck.outputs.skipped }} never ran"
```
