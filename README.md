# hiero-tck-runner

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A GitHub Action to run the **Hiero SDK Technology Compatibility Kit (TCK)** test suite against a target JSON-RPC server and network node. 

> ⚠️ **Note:** This is an unofficial GitHub Action designed to simplify SDK testing against [`hiero-ledger/hiero-sdk-tck`](https://github.com/hiero-ledger/hiero-sdk-tck).

---

## Quick Start

Add this step to your GitHub Actions workflow file (e.g., `.github/workflows/tck-tests.yml`) after starting your SDK's JSON-RPC server:

```yaml
- name: Run Hiero TCK Test Suite
  uses: manishdait/hiero-tck-runner@main
  with:
    rpcServerPort: '8544'
    tckTag: 'v0.12.0'
```

##  Inputs

| Input | Description | Required | Default |
| ----- | ----------- | -------- | --------|
| `rpcServerPort` | Target JSON-RPC server Port under test | False | `8544` |
| `nodeIp` | IP address and port of the consensus node | False | `127.0.0.1:35211` |
| `nodeAccountId` |	Account ID of the consensus node | False | `0.0.3`|
| `operatorAccountId` | Operator account ID used to sign transactions |	False |	`0.0.2`|
| `operatorPrivateKey` | Operator account private key for signing |	False |	`302e02...` (Solo Default Admin Key) |
| `mirrornodeGrpcUrl` |	Address for the mirror node gRPC service |	False |	`127.0.0.1:5600` |
| `mirrornodeRestUrl` | REST API URL for the mirror node | False |	`http://127.0.0.1:38081` |
| `mirrornodeRestJavaUrl` |	Java-based REST API URL for the mirror node | False | `http://127.0.0.1:8084` |
| `tckTag` | Git tag, branch, or commit SHA of hiero-sdk-tck | False | `v0.12.0` |
| `dockerfilePath` | Path to the Dockerfile relative to repository root | False | `./Dockerfile` |


## Usage
```yml
name: Test TCK endpoints

on:
  push:
  pull_request:

permissions:
  contents: read

jobs:
  tck-test:
    name: "Run TCK test"
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v7.0.1

      - name: Prepare Hiero Solo
        id: solo
        uses: hiero-ledger/hiero-solo-action@v0.23.0
        with:
          installMirrorNode: true

      - name: Run TCK test
        uses: manishdait/hiero-tck-runner@main
```