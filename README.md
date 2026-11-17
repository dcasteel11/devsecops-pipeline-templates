# devsecops-pipeline-templates

Reusable security gates for CI/CD, built once and consumed by every application team. The same three gates are provided for **GitHub Actions** (a reusable workflow) and **Azure DevOps** (a stage template), so organizations running both platforms enforce one standard.

| Gate | Tool | Fails the pipeline when |
|---|---|---|
| Secret scanning | Gitleaks | A credential or key is found in the code or history |
| SAST | SonarQube | The project's quality gate is red |
| SCA / open source policy | Sonatype Nexus IQ | A component violates a policy set to Fail for the stage |

## Why centralize the gates

When every team writes its own scanning steps, coverage drifts: a scan gets commented out, a gate is set to warn only, a new repo ships without one. Publishing the gates as a versioned template means platform and security teams own the controls, application teams get them with a few lines of YAML, and a policy change rolls out everywhere by updating one file.

## GitHub Actions

```yaml
jobs:
  devsecops:
    uses: dcasteel11/devsecops-pipeline-templates/.github/workflows/devsecops-scan.yml@main
    with:
      sonar-project-key: my-app
      nexus-iq-application-id: my-app
    secrets: inherit

  build:
    needs: devsecops
```

Full example: [`examples/github-caller.yml`](examples/github-caller.yml). Required secrets: `SONAR_TOKEN`, `SONAR_HOST_URL`, and for Nexus IQ `NEXUS_IQ_URL`, `NEXUS_IQ_USERNAME`, `NEXUS_IQ_PASSWORD`.

## Azure DevOps

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: dcasteel11/devsecops-pipeline-templates
      endpoint: github-connection

stages:
  - template: azure-devops/devsecops-scan.yml@templates
    parameters:
      sonarServiceConnection: sonarqube
      sonarProjectKey: my-app
      nexusIqApplicationId: my-app
```

Full example: [`examples/azure-pipelines.yml`](examples/azure-pipelines.yml). Requires the SonarQube extension and service connection, and a variable group named `nexus-iq`.

## Design choices

**Gates block, they don't warn.** The SonarQube quality gate is awaited and enforced, and Nexus IQ fails on policy, so downstream build and deploy stages never run against code that failed a control.

**Nexus IQ runs through the CLI**, not a marketplace extension, so the same command works on both platforms and on self-hosted agents.

**Least privilege.** The GitHub workflow runs with `contents: read`, and secrets are passed explicitly to the steps that need them.

**SCA is optional per repo.** Leave the Nexus IQ application ID empty to run only secret scanning and SAST, which lets teams adopt the gates in phases.

## Repository CI

Workflows are checked with actionlint, and all YAML, including the Azure DevOps templates, with yamllint.
