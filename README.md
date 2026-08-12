# Sample.WebApp — Azure DevOps CI/CD Pipelines

Technical task submission for **UINSURE**.

This repo contains a two-pipeline CI/CD setup for `Sample.WebApp`: a build
pipeline that compiles, publishes, and packages the app, and a deploy
pipeline that promotes that build through QA and Production using
slot-swap deployments, environment-based approvals, and branch-based
gating.

## Why two pipelines instead of one

The original brief was a single pipeline with a `Build` stage followed by
`DeployQA` and `DeployProd` stages. That was split into two pipelines
connected via an Azure DevOps [pipeline resource](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/resources?view=azure-devops&tabs=schema#define-a-pipelines-resource):

- **`build-pipeline.yml`** builds and publishes the artifact independently
  of any deployment target. It can be reused, run on PRs for validation,
  and versioned separately from how/where things get deployed.
- **`deploy-pipeline.yml`** doesn't build anything itself — it consumes the
  artifact from a specific build run (referenced via
  `resources.pipeline.buildPipeline`) and deploys it. This gives a clean
  separation of concerns and means a deploy can be re-run or re-targeted
  without re-running the build.

## Repo layout

```
pipelines
  build.yaml               # CI: build, test, publish, package the WebApp artifact
  deploy-webapp.yaml              # CD: deploy that artifact through QA -> Prod
  templates/
    build-template.yaml                # Reusable stage template (build _ unit tests)
    deploy-template.yaml               # Reusable stage template (deploy + swap), parameterized per environment
    set-deployment-variables.yaml        # Reusable step: pick Primary vs DR (Secondary) app service/resource group
```

## `build-pipeline.yml`

- Triggers (CI) on pushes to `main`, `release/*`, `feature/*`, and
  `hotfix/*`.
- Also runs on pull requests targeting `main` (`pr:` trigger), for build
  validation before merge.
- Single `Build` stage:
  1. `dotnet build`
  2. **`Run Unit Tests`** — `dotnet test` against `**/*Tests.csproj`
     (adjust the glob if the actual test project naming differs), with
     `.trx` results and `XPlat Code Coverage` output. A failing test
     fails this step, which stops the pipeline before an artifact is
     ever published or a tag is created.
  3. **`Publish Test Results`** / **`Publish Code Coverage`** — surface
     results in the pipeline's Tests and Code Coverage tabs, using
     `succeededOrFailed()` so results still show even when tests failed.
  4. `dotnet publish` (zipped) → `PublishPipelineArtifact@1` publishes
     the package as pipeline artifact `WebApp`.

## `deploy.yml`

- `trigger: none` — this pipeline never runs on a direct push. It only
  runs when triggered by a completed build via the `resources.pipelines`
  block, which mirrors the same branch list as the build pipeline
  (`main`, `release/*`, `feature/*`, `hotfix/*`).
- Calls the `deploy-template.yaml` template twice — once for **QA**, once for
  **Prod** — passing in the variable group, environment name, and (for
  Prod) a branch restriction.

### Deployment flow per stage

Each call to `deploy-template.yml` produces a stage with:

1. **`Set<Stage>Variables`** — runs `templates/set-deployment-vars.yml`,
   which reads the `enableDisasterRecoveryDeploy` pipeline variable and
   outputs `AppServiceName` / `AppResourceGroup` pointing at either the
   **Primary** region resources or the **Secondary** (DR) region
   resources, sourced from the stage's variable group.
2. **`DeployWebApp`** — downloads the exact build artifact that triggered
   this run (`runVersion: specific`, keyed off
   `resources.pipeline.buildPipeline.runID`, so a QA deploy from a
   feature branch can never accidentally pick up a `main` build) and
   deploys it to the App Service's `staging` slot.
3. **`SwapSlot`** — swaps `staging` into production (blue/green swap).

### Branch gating — QA vs Prod

- **QA** deploys from a build on _any_ of the CI-triggered branches
  (`main`, `release/*`, `feature/*`, `hotfix/*`) — no restriction.
- **Prod** only deploys if the triggering _build_ ran against `main`,
  `release/*`, or `hotfix/*` (feature branches are excluded), enforced via
  a `branchCondition` expression checked against
  `resources.pipeline.buildPipeline.sourceBranch`.

### Disaster recovery

`enableDisasterRecoveryDeploy` is a pipeline variable checked independently
in both the QA and Prod stages. When `true`, deploys go to the Secondary
(DR) region's App Service and resource group instead of Primary — same
mechanism, same template, in both stages.

### Approvals — Azure DevOps Environments

Manual gating before a swap is handled with [Environments](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/environments?view=azure-devops)
rather than the older `ManualValidation@0` task. Each stage uses **two**
environments:

| Job            | Environment              | Purpose                                     |
| -------------- | ------------------------ | ------------------------------------------- |
| `DeployWebApp` | `<environmentName>`      | Staging-slot deploy, tracked, no gate       |
| `SwapSlot`     | `<environmentName>-Swap` | Swap to production — approval attached here |

This repo defines `Sample-WebApp-QA` and `Sample-WebApp-Prod`, giving four
environments in total (`Sample-WebApp-QA`, `Sample-WebApp-QA-Swap`,
`Sample-WebApp-Prod`, `Sample-WebApp-Prod-Swap`).

Approval checks can't be set from YAML — they're portal-side config. After
first run (which auto-creates the environments), an **Approval** check is
added to `Sample-WebApp-Prod-Swap` only, so a human must sign off before
production goes live, while QA swaps proceed automatically.

## Setup required in Azure DevOps (not in this repo)

1. Create the build pipeline pointing at `build-pipeline.yml`, named
   `Sample.WebApp-Build` (must match the `source:` value in
   `deploy-pipeline.yml`'s `resources.pipelines` block).
2. Create the deploy pipeline pointing at `deploy-pipeline.yml`.
3. Create variable groups `Sample.WebApp-QA` and `Sample.WebApp-Prod`,
   each containing:
   - `AzureServiceConnection`
   - `PrimaryAppServiceName`, `PrimaryResourceGroup`
   - `SecondaryAppServiceName`, `SecondaryResourceGroup`
4. Set the `enableDisasterRecoveryDeploy` pipeline variable (`true`/`false`)
   on the deploy pipeline.
5. Run the deploy pipeline once to auto-create the four Environments, then
   add an Approval check to `Sample-WebApp-Prod-Swap`
   (**Pipelines → Environments → Sample-WebApp-Prod-Swap → Approvals and
   checks**).
6. Reference the `coverlet.collector` NuGet package in the test
   project(s), so `--collect:"XPlat Code Coverage"` actually produces a
   `coverage.cobertura.xml` for the Code Coverage tab to pick up.

## Design decisions worth calling out

- **Templates over duplication** — the original single-file pipeline
  repeated the QA/Prod deploy logic (and the Primary/Secondary picker
  inside it) almost verbatim. Both are now templated and parameterized,
  so a change to deploy logic happens once.
- **Exact-artifact download** — `DownloadPipelineArtifact@2` pins to the
  specific triggering build run rather than "latest from branch," which
  matters once QA can be triggered from any feature branch.
- **Two environments per stage** rather than one — keeps the approval gate
  scoped to the swap only, not the staging deploy, matching the original
  pipeline's behaviour (auto-deploy to staging, manual gate before go-live).
