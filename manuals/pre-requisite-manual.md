# Pre-Requisite Manual: Project Setup & Bootstrap

> **Generated**: 2026-02-22
> **Purpose**: Step-by-step guide for manual setup before Terraform or CI/CD can run.
> **References**: INFRA-015 (state bucket), INFRA-016 (Terraform scope), SEC-012 (Terraform SA), SEC-014 (App Deployer SA)
>
> All commands use `gcloud` CLI. Install it first: https://cloud.google.com/sdk/docs/install

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [GCP Project Setup](#2-gcp-project-setup)
3. [Enable Bootstrap APIs](#3-enable-bootstrap-apis)
4. [Terraform State Bucket](#4-terraform-state-bucket)
5. [Terraform Service Account](#5-terraform-service-account)
6. [Workload Identity Federation (WIF)](#6-workload-identity-federation-wif)
7. [App Deployer Service Account](#7-app-deployer-service-account)
8. [GitHub Repository Configuration](#8-github-repository-configuration)
9. [Firestore Enterprise Verification](#9-firestore-enterprise-verification)
10. [Domain & DNS Preparation](#10-domain--dns-preparation)
11. [MaxMind GeoIP Setup](#11-maxmind-geoip-setup)
12. [Verification Checklist](#12-verification-checklist)

---

## 1. Prerequisites

Before starting, ensure you have:

- [ ] A Google Cloud account with billing enabled
- [ ] `gcloud` CLI installed and authenticated (`gcloud auth login`)
- [ ] A GitHub account with a repository for the project (e.g., `<owner>/personal-website`)
- [ ] Domain `tjmonsi.com` registered (or your chosen domain)
- [ ] Basic familiarity with GCP IAM, Terraform, and GitHub Actions

---

## 2. GCP Project Setup

### 2.1 Create or Select a GCP Project

```bash
# Option A: Create a new project
gcloud projects create <project-id> --name="Personal Website"

# Option B: Use an existing project
gcloud config set project <project-id>
```

> Replace `<project-id>` with your actual GCP project ID throughout this guide.

### 2.2 Link Billing Account

```bash
# List available billing accounts
gcloud billing accounts list

# Link billing to project
gcloud billing projects link <project-id> --billing-account=<billing-account-id>
```

### 2.3 Set Default Region

```bash
gcloud config set compute/region asia-southeast1
```

---

## 3. Enable Bootstrap APIs

These APIs must be enabled before Terraform can run. Terraform will enable additional APIs, but these are needed for the bootstrap process itself:

```bash
gcloud services enable \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com \
  cloudresourcemanager.googleapis.com \
  storage.googleapis.com \
  serviceusage.googleapis.com \
  --project=<project-id>
```

---

## 4. Terraform State Bucket

Create the GCS bucket for Terraform remote state (INFRA-015):

```bash
# Create the bucket
gcloud storage buckets create gs://<project-id>-terraform-state \
  --project=<project-id> \
  --location=asia-southeast1 \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access

# Enable versioning (allows state rollback)
gcloud storage buckets update gs://<project-id>-terraform-state \
  --versioning
```

### Verification

```bash
gcloud storage buckets describe gs://<project-id>-terraform-state --format="value(versioning.enabled)"
# Expected: True
```

---

## 5. Terraform Service Account

Create the Terraform builder SA and grant all 18 required roles (SEC-012):

### 5.1 Create the Service Account

```bash
gcloud iam service-accounts create terraform-builder \
  --display-name="Terraform Builder SA" \
  --description="WIF-mapped SA for Terraform CI/CD pipeline (SEC-012)" \
  --project=<project-id>
```

### 5.2 Grant IAM Roles (18 Roles)

```bash
PROJECT_ID="<project-id>"
SA_EMAIL="terraform-builder@${PROJECT_ID}.iam.gserviceaccount.com"

# Grant all 18 roles
ROLES=(
  "roles/storage.admin"
  "roles/run.admin"
  "roles/cloudfunctions.admin"
  "roles/compute.networkAdmin"
  "roles/compute.securityAdmin"
  "roles/cloudscheduler.admin"
  "roles/firebasehosting.admin"
  "roles/dns.admin"
  "roles/artifactregistry.admin"
  "roles/logging.admin"
  "roles/monitoring.admin"
  "roles/iam.serviceAccountAdmin"
  "roles/iam.workloadIdentityPoolAdmin"
  "roles/iam.serviceAccountUser"
  "roles/serviceusage.serviceUsageAdmin"
  "roles/pubsub.admin"
  "roles/bigquery.admin"
  "roles/datastore.owner"
)

for ROLE in "${ROLES[@]}"; do
  gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
    --member="serviceAccount:${SA_EMAIL}" \
    --role="${ROLE}" \
    --condition=None
done
```

### Verification

```bash
gcloud projects get-iam-policy <project-id> \
  --flatten="bindings[].members" \
  --filter="bindings.members:terraform-builder@" \
  --format="table(bindings.role)" | sort | uniq | wc -l
# Expected: 18
```

---

## 6. Workload Identity Federation (WIF)

Create the WIF pool and provider that both the Terraform pipeline and the App Deployer pipeline will use (SEC-012, SEC-014). Both pipelines share `terraform-cicd-pool` since they operate from the same GitHub repository.

### 6.1 Create WIF Pool

```bash
gcloud iam workload-identity-pools create terraform-cicd-pool \
  --project=<project-id> \
  --location="global" \
  --display-name="Terraform CI/CD Pool" \
  --description="WIF pool for GitHub Actions pipelines from personal-website repo"
```

### 6.2 Create OIDC Provider

```bash
gcloud iam workload-identity-pools providers create-oidc github-actions \
  --project=<project-id> \
  --location="global" \
  --workload-identity-pool="terraform-cicd-pool" \
  --display-name="GitHub Actions" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository == '<owner>/personal-website'"
```

> Replace `<owner>` with your GitHub username or organization.

### 6.3 Bind Terraform SA to WIF Pool

```bash
PROJECT_NUMBER=$(gcloud projects describe <project-id> --format="value(projectNumber)")

gcloud iam service-accounts add-iam-policy-binding \
  terraform-builder@<project-id>.iam.gserviceaccount.com \
  --project=<project-id> \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/terraform-cicd-pool/attribute.repository/<owner>/personal-website"
```

### Verification

```bash
# Verify pool exists
gcloud iam workload-identity-pools describe terraform-cicd-pool \
  --project=<project-id> \
  --location="global" \
  --format="value(name)"

# Verify provider exists
gcloud iam workload-identity-pools providers describe github-actions \
  --project=<project-id> \
  --location="global" \
  --workload-identity-pool="terraform-cicd-pool" \
  --format="value(name)"
```

---

## 7. App Deployer Service Account

Create the App Deployer SA that the application CI/CD pipeline will use (SEC-014). Its roles will be managed by Terraform later, but the SA itself and its WIF binding need to exist so we can configure GitHub secrets.

### 7.1 Create the Service Account

```bash
gcloud iam service-accounts create app-deployer \
  --display-name="App Deployer SA" \
  --description="WIF-mapped SA for application CI/CD pipeline (SEC-014)" \
  --project=<project-id>
```

### 7.2 Bind App Deployer SA to WIF Pool

The App Deployer SA shares the same `terraform-cicd-pool` since it deploys from the same repository:

```bash
PROJECT_NUMBER=$(gcloud projects describe <project-id> --format="value(projectNumber)")

gcloud iam service-accounts add-iam-policy-binding \
  app-deployer@<project-id>.iam.gserviceaccount.com \
  --project=<project-id> \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/terraform-cicd-pool/attribute.repository/<owner>/personal-website"
```

> **Note**: The App Deployer SA's 5 deployment roles (`roles/artifactregistry.writer`, `roles/run.developer`, `roles/firebasehosting.admin`, `roles/cloudfunctions.developer`, `roles/iam.serviceAccountUser`) will be granted by Terraform in Phase 1. For the bootstrap phase, only the WIF binding is needed.

---

## 8. GitHub Repository Configuration

### 8.1 Get WIF Provider Resource Name

```bash
# Get the full provider resource name (needed for GitHub Actions)
gcloud iam workload-identity-pools providers describe github-actions \
  --project=<project-id> \
  --location="global" \
  --workload-identity-pool="terraform-cicd-pool" \
  --format="value(name)"

# Output format:
# projects/<project-number>/locations/global/workloadIdentityPools/terraform-cicd-pool/providers/github-actions
```

Save this value — it's the `WIF_PROVIDER` secret.

### 8.2 Configure GitHub Actions Secrets

Navigate to your GitHub repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:

| Secret Name | Value | Description |
|-------------|-------|-------------|
| `GCP_PROJECT_ID` | `<project-id>` | Your GCP project ID |
| `WIF_PROVIDER` | `projects/<number>/locations/global/workloadIdentityPools/terraform-cicd-pool/providers/github-actions` | Full WIF provider resource name |
| `TERRAFORM_SA_EMAIL` | `terraform-builder@<project-id>.iam.gserviceaccount.com` | Terraform SA email |
| `APP_DEPLOYER_SA_EMAIL` | `app-deployer@<project-id>.iam.gserviceaccount.com` | App Deployer SA email |
| `MAXMIND_LICENSE_KEY` | *(from Step 11)* | MaxMind GeoIP license key |

### 8.3 Configure GitHub Actions Environment (Optional but Recommended)

For the Terraform Apply step requiring manual approval:

1. Go to **Settings** → **Environments** → **New environment**
2. Create environment: `production`
3. Enable **Required reviewers** and add yourself
4. The Terraform CI/CD workflow will reference this environment for the Apply stage

---

## 9. Firestore Enterprise Verification

Firestore Enterprise (MongoDB compatibility mode) may not be fully supported by Terraform's `google_firestore_database` resource. Verify during implementation:

### 9.1 Check Terraform Support

```bash
# Check if google_firestore_database supports MongoDB compat mode
# Reference: https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/firestore_database
# Look for `type` or `database_type` attribute supporting "FIRESTORE_NATIVE" vs enterprise/MongoDB mode
```

### 9.2 If Terraform Does NOT Support It — Manual Creation

```bash
# Use the Firestore Enterprise console or gcloud to create the database
# Navigate to: https://console.cloud.google.com/firestore
# Select: Firestore Enterprise (MongoDB compatibility)
# Region: asia-southeast1
# Database ID: (default)
```

### 9.3 Create Firestore Native Database (for vector search)

This is a separate database that Terraform should be able to create. If not:

```bash
gcloud firestore databases create \
  --database=vector-search \
  --location=asia-southeast1 \
  --type=firestore-native \
  --project=<project-id>
```

---

## 10. Domain & DNS Preparation

### 10.1 Register Domain

Ensure `tjmonsi.com` (or your domain) is registered with a domain registrar.

### 10.2 Note Current Nameservers

After Terraform creates the Cloud DNS zone, you'll need to update your domain's nameservers.

```bash
# After Terraform creates the zone, get the nameservers:
gcloud dns managed-zones describe <zone-name> \
  --project=<project-id> \
  --format="value(nameServers)"
```

> **Important**: Nameserver update happens AFTER Terraform Phase 1 is complete. This step is just preparation.

---

## 11. MaxMind GeoIP Setup

The backend Docker image bundles a MaxMind GeoLite2-Country database for IP geolocation.

### 11.1 Create a MaxMind Account

1. Go to https://www.maxmind.com/en/geolite2/signup
2. Create a free account
3. Navigate to **Account** → **Manage License Keys** → **Generate New License Key**
4. Save the license key as `MAXMIND_LICENSE_KEY` in GitHub Actions secrets (Step 8.2)

### 11.2 Verify Download Works

```bash
# Test the download URL (replace with your key)
curl -o /dev/null -w "%{http_code}" \
  "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&license_key=<your-key>&suffix=tar.gz"
# Expected: 200
```

---

## 12. Verification Checklist

Run through this checklist to confirm all prerequisites are complete:

### GCP Project

- [ ] Project exists and billing is linked
- [ ] `gcloud config get-value project` returns `<project-id>`

### Bootstrap APIs

```bash
gcloud services list --enabled --project=<project-id> --filter="NAME:(iam OR iamcredentials OR sts OR cloudresourcemanager OR storage OR serviceusage)" --format="value(NAME)" | sort
```

Expected output (6 APIs):
```
cloudresourcemanager.googleapis.com
iam.googleapis.com
iamcredentials.googleapis.com
serviceusage.googleapis.com
storage.googleapis.com
sts.googleapis.com
```

### Terraform State Bucket

```bash
gcloud storage buckets describe gs://<project-id>-terraform-state --format="json(versioning,location)" 2>/dev/null && echo "OK" || echo "MISSING"
```

### Service Accounts

```bash
gcloud iam service-accounts list --project=<project-id> --filter="email:(terraform-builder@ OR app-deployer@)" --format="table(email,displayName)"
```

Expected: 2 service accounts listed.

### Terraform SA Roles

```bash
gcloud projects get-iam-policy <project-id> \
  --flatten="bindings[].members" \
  --filter="bindings.members:terraform-builder@<project-id>.iam.gserviceaccount.com" \
  --format="value(bindings.role)" | sort | wc -l
```

Expected: `18`

### WIF Pool & Provider

```bash
gcloud iam workload-identity-pools list --project=<project-id> --location=global --format="value(name)"
# Expected: contains "terraform-cicd-pool"

gcloud iam workload-identity-pools providers list \
  --project=<project-id> \
  --location=global \
  --workload-identity-pool=terraform-cicd-pool \
  --format="value(name)"
# Expected: contains "github-actions"
```

### WIF Bindings

```bash
# Terraform SA binding
gcloud iam service-accounts get-iam-policy terraform-builder@<project-id>.iam.gserviceaccount.com \
  --format="json(bindings)" | grep -c "workloadIdentityUser"
# Expected: 1

# App Deployer SA binding
gcloud iam service-accounts get-iam-policy app-deployer@<project-id>.iam.gserviceaccount.com \
  --format="json(bindings)" | grep -c "workloadIdentityUser"
# Expected: 1
```

### GitHub Secrets

Verify in GitHub → Settings → Secrets → Actions:

- [ ] `GCP_PROJECT_ID` — set
- [ ] `WIF_PROVIDER` — set (full resource name)
- [ ] `TERRAFORM_SA_EMAIL` — set
- [ ] `APP_DEPLOYER_SA_EMAIL` — set
- [ ] `MAXMIND_LICENSE_KEY` — set

### MaxMind

```bash
curl -s -o /dev/null -w "%{http_code}" "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&license_key=<your-key>&suffix=tar.gz"
# Expected: 200
```

---

## Quick Reference: Resource Summary

| Resource | Name/ID | Creation Method |
|----------|---------|-----------------|
| GCP Project | `<project-id>` | Manual |
| State Bucket | `gs://<project-id>-terraform-state` | Manual |
| Terraform SA | `terraform-builder@<project-id>.iam.gserviceaccount.com` | Manual (18 roles) |
| App Deployer SA | `app-deployer@<project-id>.iam.gserviceaccount.com` | Manual (shell only, roles via Terraform) |
| WIF Pool | `terraform-cicd-pool` | Manual |
| WIF Provider | `github-actions` (OIDC) | Manual |
| Attribute Condition | `assertion.repository == "<owner>/personal-website"` | Manual |

**Everything else is managed by Terraform (Phase 1).**

---

## Next Steps

Once all checkboxes in the verification checklist are green:

1. Proceed to **Phase 1** in [tasks.md](../tasks.md) — Terraform Foundation
2. Initialize the Terraform project and run the first `terraform plan`
3. After Terraform Apply, update domain nameservers to Cloud DNS
