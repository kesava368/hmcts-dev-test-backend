# HMCTS Dev Test Backend
This will be the backend for the brand new HMCTS case management system. As a potential candidate we are leaving
this in your hands. Please refer to the brief for the complete list of tasks! Complete as much as you can and be
as creative as you want.

You should be able to run `./gradlew build` to start with to ensure it builds successfully. Then from that you
can run the service in IntelliJ (or your IDE of choice) or however you normally would.

There is an example endpoint provided to retrieve an example of a case. You are free to add/remove fields as you
wish.

## Running Locally with Docker Compose

### 1. Environment Setup

Set your database and application environment variables in your terminal:

```bash
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=devtest
export DB_USER_NAME=postgres
export DB_PASSWORD=secure_password
export APP_PORT=4000
```

### 2. Start the Application

Open your terminal in the project root directory and run:

docker compose up -d

### 3. Verify Application Health and Endpoints

#### Welcome message
curl http://localhost:4000/

#### Health check (includes PostgreSQL connection status)
curl http://localhost:4000/health

#### Sample case JSON endpoint
curl http://localhost:4000/get-example-case

### 4. Stopping containers cleanly

#### Stop containers and wipe the persistent pgdata volume
docker compose down -v

### 5. Infrastructure setup using real Azure environment

For production deployments, Terraform state should be stored remotely in a centralized, secure location to enable team collaboration, state locking, and encryption at rest.

#### Recommended Configuration:
State can be stored in an Azure Blob Storage container using the `azurerm` backend block (see `providers.tf`):

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "myapp-tfstate-rg"
    storage_account_name = "myapptfstatesa"
    container_name       = "tfstate"
    key                  = "dev.terraform.tfstate"
  }
}
```

- **Creation of Storage account in Azure:** State files are stored in an encrypted Azure Blob Storage container (myapptfstatesa/tfstate) with automatic lease locking to ensure safe, concurrent pipeline execution.
- **Keyless OIDC Authentication:** Service Principal creation with federated credentials to secure Azure to GitHub zero trust integration.
- **Automated PR-Driven Deployment:** Pull requests trigger terraform plan and post an interactive Markdown comment on the PR; merging to master automatically runs terraform apply against your Azure Container App, PostgreSQL, and Key Vault resources.

### 6. Image Tagging Strategy Rationale

**Git Commit SHA** is used as our primary tagging strategy for this implementation, with a plan to incorporate **Semantic Versioning (SemVer)** for production releases.

Release notes needs to be incorporated irrespective of any strategy chosen to ensure we have complete picture of what's deployed.

#### Chosen Strategy: Git Commit SHA (`sha-<commit>`)
* **Traceability:** Establishes a direct, immutable 1:1 mapping between the running container image and the exact Git commit in source control.
* **Collision-Free:** Prevents tag overwrites when multiple feature branches are built and tested concurrently.

#### Future Strategy: Semantic Versioning (`vX.Y.Z`)
For production releases, Git release tags following Semantic Versioning will be used to clearly communicate release intent:
* **Major (`X.0.0`):** Bumps for breaking changes or major architectural updates.
* **Minor (`1.Y.0`):** Bumps for new, backwards-compatible features.
* **Patch (`1.0.Z`):** Bumps for backwards-compatible bug fixes and security hotfixes.

#### Rejected Alternative: `branch-buildnumber` (e.g., `feature-xyz-42`)
* **Short-Lived Branches:** Because feature branches are deleted immediately upon merging into `master`, using branch names leads to orphaned registry tags tied to non-existent branches.
* **Registry Clutter:** Generates unnecessary tag accumulation in the container registry (GHCR), adding overhead for cleanup and lifecycle management.


### 7. High-Level CI/CD Pipeline Overview

- This pipeline automates code quality checks, infrastructure validation, security scanning, and container delivery.
- It enforces automated quality gates before any code can be merged into `master` or pushed to the GitHub Container Registry (GHCR).

#### Pipeline Execution Strategy (Path Filtering)

To optimize execution time and resource usage, the pipeline uses path filtering to run only the jobs relevant to the files modified in a pull request or commit.

#### Pipeline Stages & Job Breakdown

#### Stage 1: Path Filter (`changes`)
* **Purpose:** Detects whether code changes belong to infrastructure (`infrastructure/**`) or application code.
* **Key Actions:** Evaluates changed paths to conditionally trigger subsequent pipeline jobs, preventing unnecessary builds.

#### Stage 2: Infrastructure Validation (`terraform-checks`)
* **Trigger:** Runs only when changes are made inside the `infrastructure/` directory.
* **Key Actions:**
  * **Format Check (`terraform fmt`):** Ensures code conforms to standard Terraform layout and syntax.
  * **Validation (`terraform validate`):** Verifies configuration syntax and internal consistency without initializing remote state.

#### Stage 3: Application Build & Test (`build-and-test`)
* **Trigger:** Runs on application code changes.
* **Key Actions:**
  * **Setup & Optimization:** Configures JDK 21 and provisions Gradle via `setup-gradle` for optimized caching and execution.
  * **Static Analysis & Testing (`./gradlew check`):** Runs unit tests, integration tests, and static code analysis (Checkstyle) in a single pass.
  * **Artifact Archiving:** Saves test results and code quality reports as downloadable GitHub artifacts.

#### Stage 4: Container Build, Security Scan & Publishing (`container-ci`)
* **Trigger:** Runs on application code changes after Stage 3 successfully passes.
* **Key Actions:**
  1. **Metadata Tagging:** Generates a short Git commit SHA tag (`sha-<commit>`) and attaches OCI repository metadata to link the image directly to the GitHub repository package page.
  2. **Security Gates (Trivy Scan):**
  * **HIGH Severity Scan:** Non-blocking warning scan to report potential vulnerabilities.
  * **CRITICAL Severity Scan:** Hard-blocking quality gate that fails the pipeline if critical vulnerabilities are detected.
  3. **Publishing (GHCR):** Pushes the validated container image to GitHub Container Registry **only on branch merges** (skips push step on PRs).


### 8. Trade offs and future improvements

- **Layered architecture**: Decoupling single state file into distinct layers (e.g., 01-networking, 02-database, 03-app-runtime) and tf code module wise separation.
- **Semantic versioning**: Tagging using semantic versions for GHCR images.
- **Documentation** - Making it more concise instead of current long descriptions.
- **CI improvements** - Improve CI workflow (image scan particularly).
- **Git ignore files** - Can be improved.
