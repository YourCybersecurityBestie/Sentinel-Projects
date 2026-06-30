# Defender for Cloud CLI with GHEC Data Residency

This guide shows how to use Defender for Cloud CLI from GitHub Enterprise Cloud (GHEC) with data residency (`ghe.com`) to scan container images and ingest results into Microsoft Defender for Cloud using token-based authentication.

## Why this pattern

For some GHEC data residency environments, the standard Defender for Cloud GitHub connector path is constrained. This runbook uses a connectorless ingestion pattern:

- Pipeline scan in GitHub Actions
- Token-based auth to Defender for Cloud DevOps ingestion
- Findings uploaded to Defender for Cloud
- Optional code-to-runtime enrichment using image provenance metadata

## What this gives you

- Container image scanning from CI/CD
- Findings ingested into Defender for Cloud recommendations/context
- A path that does not require GitHub connector auth in CI

## What this does not automatically give you

- Full connector-dependent DevOps inventory/posture experiences
- Connector-dependent bidirectional sync experiences

## Prerequisites

1. Azure subscription onboarded to Defender for Cloud.
2. Defender CSPM enabled (recommended).
3. Container workload path in scope (for example ACR plus AKS) if you want runtime correlation.
4. Security Admin role to create DevOps Ingestion credentials.
5. GitHub Actions workflow building or referencing container images.

## Step 1: Create Defender DevOps Ingestion credentials

In Azure portal:

1. Open **Microsoft Defender for Cloud**.
2. Go to **Management -> Environment settings -> Integrations**.
3. Select **Add integration -> DevOps Ingestion (Preview)**.
4. Enter an app name.
5. Choose tenant, expiration, and enable token.
6. Save.
7. Copy and securely store:
   - Tenant ID
   - Client ID
   - Client Secret

Important: The secret is only shown once.

## Step 2: Configure secure secret delivery (enterprise)

Do not store credentials in code, workflow files, artifacts, or plaintext variables.

Use one of these patterns (in priority order):

1. Preferred: OIDC from GitHub Actions to your secret manager, then fetch credentials at runtime.
2. Good: GitHub Environment secrets with required reviewers and branch protections.
3. Acceptable fallback: GitHub Organization secrets scoped only to approved repositories.

Avoid repository-level secrets for strict enterprise environments unless there is an explicit exception.

Required runtime values:

- `DEFENDER_TENANT_ID`
- `DEFENDER_CLIENT_ID`
- `DEFENDER_CLIENT_SECRET`

## Step 3: Add Defender CLI scanning to workflow

Add these steps to your GitHub Actions workflow job.

```yaml
name: defender-cli-image-scan

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

jobs:
  scan:
    runs-on: ubuntu-latest
    environment: production
    env:
      IMAGE_NAME: myregistry.azurecr.io/myapp:${{ github.sha }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Download Defender for Cloud CLI
        run: |
          curl -L -o defender "https://aka.ms/defender-cli_linux-x64"
          chmod +x defender

      - name: Run Defender for Cloud CLI scan and ingest
        run: |
          ./defender scan image "${IMAGE_NAME}"
        continue-on-error: true
        env:
          DEFENDER_TENANT_ID: ${{ secrets.DEFENDER_TENANT_ID }}
          DEFENDER_CLIENT_ID: ${{ secrets.DEFENDER_CLIENT_ID }}
          DEFENDER_CLIENT_SECRET: ${{ secrets.DEFENDER_CLIENT_SECRET }}
```

Notes:

- Replace `IMAGE_NAME` with your actual built image reference.
- Set `continue-on-error: false` if you want to hard-fail the pipeline on scan issues.
- Keep this job in a protected environment with approval gates for production.

### Optional: OIDC + Azure Key Vault runtime retrieval example

Use this pattern if your enterprise policy requires secret retrieval at runtime from Key Vault.

```yaml
name: defender-cli-image-scan-oidc

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  scan:
    runs-on: ubuntu-latest
    environment: production
    env:
      IMAGE_NAME: myregistry.azurecr.io/myapp:${{ github.sha }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Azure login with OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_FEDERATED_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Fetch Defender credentials from Key Vault
        id: kv
        uses: azure/CLI@v2
        with:
          inlineScript: |
            echo "DEFENDER_TENANT_ID=$(az keyvault secret show --vault-name <kv-name> --name DEFENDER-TENANT-ID --query value -o tsv)" >> $GITHUB_ENV
            echo "DEFENDER_CLIENT_ID=$(az keyvault secret show --vault-name <kv-name> --name DEFENDER-CLIENT-ID --query value -o tsv)" >> $GITHUB_ENV
            echo "DEFENDER_CLIENT_SECRET=$(az keyvault secret show --vault-name <kv-name> --name DEFENDER-CLIENT-SECRET --query value -o tsv)" >> $GITHUB_ENV

      - name: Download Defender for Cloud CLI
        run: |
          curl -L -o defender "https://aka.ms/defender-cli_linux-x64"
          chmod +x defender

      - name: Run Defender for Cloud CLI scan and ingest
        run: |
          ./defender scan image "${IMAGE_NAME}"
        continue-on-error: true
        env:
          DEFENDER_TENANT_ID: ${{ env.DEFENDER_TENANT_ID }}
          DEFENDER_CLIENT_ID: ${{ env.DEFENDER_CLIENT_ID }}
          DEFENDER_CLIENT_SECRET: ${{ env.DEFENDER_CLIENT_SECRET }}
```

Key implementation notes:

- Configure a federated credential on the Azure app registration used by `azure/login`.
- Grant least privilege Key Vault access to only required secrets.
- Use environment approvals for production workflows.
- Rotate Defender ingestion secrets on a defined schedule.

## Step 4: (Recommended) Add provenance for stronger code-to-runtime mapping

For richer mapping in Defender for Cloud, add OCI labels during image build and deploy by digest.

Recommended labels:

- `org.opencontainers.image.source`
- `org.opencontainers.image.revision`
- `org.opencontainers.image.url`
- `org.opencontainers.image.version`
- `org.opencontainers.image.created`

Optional but recommended: enable GitHub artifact attestations for stronger provenance.

## Step 5: Validate ingestion and mapping

After a pipeline run:

1. Wait 15-30 minutes for findings ingestion.
2. In Defender for Cloud, check **Recommendations** for container findings.
3. In **Cloud Security Explorer**, validate container image relationships.
4. If using runtime (for example AKS), validate Source -> CI/CD -> Registry -> Runtime context appears over time.

## Step 6: Security hardening for token-based auth

Token-based auth is secure when operated with controls:

1. Least privilege scope.
2. Short secret expiry.
3. Rotation process (automated where possible).
4. Separate credentials per environment.
5. Prefer OIDC plus external vault retrieval at runtime; if not available, use GitHub Environment or Organization secrets with strict scope.
6. Monitor token usage and set incident revocation playbook.
7. Use protected branches, required reviewers, and environment approvals for workflow changes.
8. Restrict runner trust boundary (hardened hosted or dedicated self-hosted runners).
9. Mask secrets in logs and disable shell debug flags that could echo env values.

## Troubleshooting

1. No findings in Defender:
  - Confirm runtime credentials are injected correctly (from vault or scoped GitHub secrets).
   - Confirm subscription is onboarded and CSPM is enabled.
   - Check workflow logs for auth errors.

2. Mapping is incomplete:
   - Ensure OCI labels exist in pushed image.
   - Ensure image deployed to runtime is the same digest from CI.
   - Ensure registry/runtime are discoverable by Defender.

3. Delayed visibility:
   - Ingestion and mapping are not always immediate. Recheck after additional processing time.

## Public documentation

Core setup and auth:

- CI/CD pipeline scanning with Defender CLI:
  - https://learn.microsoft.com/azure/defender-for-cloud/ci-cd-pipeline-scanning-with-defender-cli
- Defender CLI authentication:
  - https://learn.microsoft.com/azure/defender-for-cloud/defender-cli-authentication
- Defender CLI syntax:
  - https://learn.microsoft.com/azure/defender-for-cloud/defender-cli-syntax

Mapping and validation:

- Container image code-to-runtime mapping:
  - https://learn.microsoft.com/azure/defender-for-cloud/container-image-mapping
- Code-to-runtime enrichment:
  - https://learn.microsoft.com/azure/defender-for-cloud/code-to-runtime-mapping
- Cloud Security Explorer:
  - https://learn.microsoft.com/azure/defender-for-cloud/how-to-manage-cloud-security-explorer

Platform support boundaries:

- DevOps support and prerequisites:
  - https://learn.microsoft.com/azure/defender-for-cloud/devops-support

Provenance references:

- GitHub artifact attestations:
  - https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations
- Docker metadata action:
  - https://github.com/docker/metadata-action
- OCI annotations spec:
  - https://github.com/opencontainers/image-spec/blob/main/annotations.md

## Suggested email link text

Use this text in customer communications:

"Step-by-step implementation guide: Defender for Cloud CLI with GHEC data residency"
