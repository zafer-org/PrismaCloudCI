# PrismaCloudCI

Organization-level required workflow for scanning container images with **Prisma Cloud** on every pull request.

This repository hosts a shared GitHub Actions workflow that is enforced via an organization ruleset. Any pull request in a targeted repository triggers a build of the container image and a scan using the [Prisma Cloud Scan](https://github.com/marketplace/actions/prisma-cloud-scan) action, blocking merges when vulnerabilities or compliance violations exceed the configured thresholds.

## Objective

Enforce, at the organization level, that every pull request touching a repository with a Dockerfile runs a Prisma Cloud image scan before it can be merged.

## Pre-requisites

- A Prisma Cloud user with permission to create roles and service accounts.
- A GitHub user with administrator access to the organization.

## Process overview

| Step | What | Where |
| --- | --- | --- |
| 1 | Create Prisma Cloud role + service account | Prisma Cloud console |
| 2 | Store credentials as org secrets | GitHub org settings |
| 3 | Host the reusable workflow file | This repository |
| 4 | Share the repository with the organization | This repository's settings |
| 5 | Verify allowed actions | GitHub org settings |
| 6 | Create the organization ruleset | GitHub org settings |
| 7 | (Optional) Override image name / Dockerfile per repo | Target repository variables |

---

## Step 1 — Create the Prisma Cloud service account

In Prisma Cloud, go to **Settings > Access Control > Add > Role**:

- **Name**: `CI Scanning Role` (recommended)
- **Description**: any
- **Permission Group**: `Build and Deploy Security`
- **Advanced options > Access key only**: checked

Then **Add > Service Account**:

- **Name**: `CI Scanning SA` (recommended)
- **Role**: the role created above
- **Access Key name**: your organization name (recommended)
- **Expiration Date**: your choice

Download the CSV with the access key and secret — you will need them in Step 2.

## Step 2 — Create organization secrets

In GitHub: **Organization > Settings > Secrets and variables > Actions**. Create three organization secrets:

| Secret | Value |
| --- | --- |
| `PCC_URL` | Prisma Cloud Compute URL. Found in **Runtime Security > Manage > System > Utilities > Path to Console**. |
| `PCC_USER` | Access Key ID from the service account CSV. |
| `PCC_PASS` | Secret Key from the service account CSV. |

> If you use a different secrets manager, update [prisma-image-scan.yml](.github/workflows/prisma-image-scan.yml) accordingly.

## Step 3 — The shared workflow

The workflow lives at [.github/workflows/prisma-image-scan.yml](.github/workflows/prisma-image-scan.yml). It:

1. Checks out the repository under test.
2. Derives the image name (`<owner>/<repo>:<source-branch>`, lowercased) unless the target repo defines an `IMAGE_NAME` variable.
3. Injects GitHub context as Docker `LABEL` directives into the Dockerfile (repository, actor, run IDs, branches, etc.) so the resulting image is traceable in the Prisma Cloud console.
4. Builds the image.
5. Scans it with `PaloAltoNetworks/prisma-cloud-scan@v1.6.7`.

The SARIF upload step is included but commented out — enable it only if your subscription has **GitHub Advanced Security** and you want results surfaced as code scanning alerts. See the [GitHub docs on uploading SARIF](https://docs.github.com/en/code-security/code-scanning/integrating-with-code-scanning/uploading-a-sarif-file-to-github). Enabling it also requires granting the job `security-events: write` permission.

**Repository visibility notes** (from GitHub's required-workflow rules):
- Public repo → can be required for any repo in the org.
- Private repo → can only be required for other private repos.
- Internal repo → can be required for internal and private repos.

## Step 4 — Share the repository with the organization

In this repository: **Settings > Actions > General > Access** — grant access from your organization (or enterprise, if you want this workflow available to multiple orgs).

## Step 5 — Verify allowed actions at the org level

**Organization > Settings > Actions > General**. In the *Policies* section, confirm the following actions are permitted under your current setup:

- `actions/checkout@v3`
- `PaloAltoNetworks/prisma-cloud-scan@v1.6.7`
- `github/codeql-action/upload-sarif@v3` *(only if you enable the SARIF upload step)*

If any of these is blocked, the workflow will fail to run.

## Step 6 — Create the organization ruleset

**Organization > Settings > Repository > Rulesets > New ruleset > New branch ruleset**:

- **Ruleset name**: any
- **Enforcement status**: `Active`
- **Target repositories**: any repository with a Dockerfile
- **Branch rules**:
  - ✅ **Require workflows to pass before merging**
  - ⬜ **Do not require workflows on creation** (leave unchecked so brand-new PRs are gated too)
  - **Workflow configurations > Add workflow**:
    - **Repository**: this repository
    - **Branch**: `main` (or your chosen default)
    - **Workflow file**: `.github/workflows/prisma-image-scan.yml`

Any other rule parameters can be set to taste.

## Step 7 — (Optional) Overrides per repository

By default the scanned image is named:

```
<repository-name>:<source-branch>
```

To override, set these **repository variables** in the target repo (**Settings > Secrets and variables > Actions > Variables**):

| Variable | Default | Purpose |
| --- | --- | --- |
| `IMAGE_NAME` | `<repo>:<head_ref>` (lowercased) | Override the built image tag |
| `DOCKERFILE` | `Dockerfile` | Override the Dockerfile path |

## Troubleshooting

Once everything is in place, every pull request against a targeted repo runs the CI scan. If the scan fails, likely causes are:

1. The image has vulnerabilities or compliance violations above the configured Prisma Cloud CI policy thresholds.
2. The organization secrets (`PCC_URL`, `PCC_USER`, `PCC_PASS`) are missing or incorrect.
3. One of the actions used by the workflow is not on the org's allow-list (see Step 5).
4. The `Dockerfile` path or `IMAGE_NAME` override is misconfigured on the target repository.
