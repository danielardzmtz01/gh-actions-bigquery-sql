# Automated BigQuery SQL Execution with GitHub Actions

This repository features a GitHub Action that automates the execution of SQL scripts in Google BigQuery. It triggers whenever changes are detected in specific SQL files, ensuring your BigQuery schemas and data definitions are always in sync with your codebase.

-----

## 🚀 How It Works

This GitHub Action listens for `push` events to the `main` branch. If any changes in the push affect `.sql` files within the `views/ddls/` directory (or its subfolders), the workflow will:

1.  **Checkout the repository**.
2.  **Set up the Google Cloud SDK**.
3.  **Securely authenticate to Google Cloud** using **Workload Identity Federation**, leveraging **GitHub repository variables (Vars)** and **secrets (Secrets)** for enhanced security.
4.  **Identify SQL files** that have been added, modified, or renamed within the `views/ddls/` path.
5.  **Execute each of these SQL scripts** in BigQuery using the `bq query` command.

This setup is ideal for managing BigQuery views, tables, and other objects defined via DDL (Data Definition Language) directly from your version control system.

-----

## 🛠️ GitHub Action Configuration

Below is the YAML code for the GitHub Action. You should save this in your repository under the path `.github/workflows/execute-bigquery-sql.yml` (or a similar name).

**Important Note:** This version of the action utilizes **GitHub Vars and Secrets** to store sensitive and configurable values like your GCP Project ID, Service Account email, and Workload Identity Provider details. Make sure to configure them in the settings of your GitHub repository.

```yaml
name: Execute SQL on BigQuery on file change

on:
  push:
    branches:
      - main
    paths:
      - 'views/ddls/**/*.sql'

jobs:
  run-bigquery-query:
    runs-on: ubuntu-latest
    permissions:
      contents: 'read'
      id-token: 'write' # REQUIRED for Workload Identity Federation

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Google Cloud SDK
        uses: google-github-actions/setup-gcloud@v2
        with:
          # Using a GitHub repository variable for the Project ID
          project_id: ${{ vars.PROJECT_ID }}
          version: latest
          skip_install: false

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          # Using a GitHub repository secret for the Workload Identity Provider ID
          workload_identity_provider: ${{ secrets.WIP_ID }}
          # Using a GitHub repository variable for the Service Account email
          service_account: ${{ vars.SERVICE_ACCT }}

      - name: Get modified SQL files
        id: changed-files
        uses: tj-actions/changed-files@v40
        with:
          files: 'views/ddls/**/*.sql'

      - name: Execute SQL queries in BigQuery
        if: steps.changed-files.outputs.any_changed == 'true'
        run: |
          CHANGED_FILES=""
          if [ -n "${{ steps.changed-files.outputs.added_files }}" ]; then
            CHANGED_FILES="${CHANGED_FILES} ${{ steps.changed-files.outputs.added_files }}"
          fi
          if [ -n "${{ steps.changed-files.outputs.modified_files }}" ]; then
            CHANGED_FILES="${CHANGED_FILES} ${{ steps.changed-files.outputs.modified_files }}"
          fi
          if [ -n "${{ steps.changed-files.outputs.renamed_files }}" ]; then
            CHANGED_FILES="${CHANGED_FILES} ${{ steps.changed-files.outputs.renamed_files }}"
          fi

          CHANGED_FILES=$(echo $CHANGED_FILES | xargs)

          if [ -n "$CHANGED_FILES" ]; then
            echo "Processing the following changed SQL files:"
            echo "$CHANGED_FILES"
            for file in $CHANGED_FILES; do
              echo "Executing query from: $file in BigQuery..."
              # Using the GitHub repository variable for the Project ID
              bq query --batch --project_id=${{ vars.PROJECT_ID }} --nouse_legacy_sql < "$file"
              if [ $? -ne 0 ]; then
                echo "Error executing query from $file. Check BigQuery logs."
                exit 1
              fi
              echo "Query from $file executed successfully."
            done
          else
            echo "No SQL files (added, modified, or renamed) found in 'views/ddls/' that were changed. Skipping query execution."
          fi
```

-----

## 🔒 Google Cloud Authentication Setup (Workload Identity Federation)

To enable your GitHub Action to interact with Google Cloud securely without long-lived service account keys, you'll use **Workload Identity Federation**. This allows your GitHub repository to act as an identity provider for Google Cloud.

Additionally, for this specific GitHub Action version, you'll need to configure **GitHub Variables and Secrets** in your repository.

### GitHub Variables and Secrets Setup

In your GitHub repository, navigate to **`Settings` \> `Variables` \> `Actions`** to set up **Vars** and **`Settings` \> `Secrets` \> `Actions`** to set up **Secrets**.

Add the following **Repository Variables (Vars)**:

  * **`PROJECT_ID`**: Your Google Cloud project ID (e.g., `my-gcp-project-123`).
  * **`SERVICE_ACCT`**: The full email address of the Google Cloud service account you'll use (e.g., `github-actions-sa@my-gcp-project-123.iam.gserviceaccount.com`).

Add the following **Repository Secret (Secret)**:

  * **`WIP_ID`**: The full ID of your Workload Identity Provider. This is a sensitive value that includes your project and provider information (e.g., `projects/123456789/locations/global/workloadIdentityPools/github/providers/my-repo`).

### Google Cloud Configuration Steps

The Workload Identity Federation configuration steps in Google Cloud provided below are directly sourced from the official `google-github-actions/auth` repository. For the most detailed and up-to-date information, please refer to the official repository: [https://github.com/google-github-actions/auth](https://github.com/google-github-actions/auth).

#### 0\. (Optional) Create a Google Cloud Service Account

If you don't already have one, create a dedicated Service Account for this integration. Make sure to note its email address, as you'll need it for the `SERVICE_ACCT` GitHub Variable.

```bash
# TODO: Replace ${PROJECT_ID} with your GCP Project ID.
gcloud iam service-accounts create "github-actions-sa" \
  --project "${PROJECT_ID}"
```

#### 1\. Create a Workload Identity Pool

This is the main container for your external identities.

```bash
# TODO: Replace ${PROJECT_ID} with your GCP Project ID.
gcloud iam workload-identity-pools create "github" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --display-name="GitHub Actions Pool"
```

#### 2\. Get the Full ID of the Workload Identity Pool

You'll need this full ID for the next steps.

```bash
# TODO: Replace ${PROJECT_ID} with your GCP Project ID.
gcloud iam workload-identity-pools describe "github" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --format="value(name)"
```

The resulting value will be in the format: `projects/123456789/locations/global/workloadIdentityPools/github`.

#### 3\. Create a Workload Identity Provider in that Pool

This provider enables the pool to trust OIDC tokens issued by GitHub.

🛑 **SECURITY CAUTION\!** Always add an **Attribute Condition** to restrict access to the Workload Identity Pool. This is crucial for security. A common best practice is to restrict access by your GitHub organization.

```bash
# TODO: Replace ${PROJECT_ID} with your GCP Project ID.
# TODO: Replace ${GITHUB_ORG} with your GitHub organization name (e.g., "my-company-org").
gcloud iam workload-identity-pools providers create-oidc "my-repo" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github" \
  --display-name="My GitHub repo Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
  --attribute-condition="assertion.repository_owner == '${GITHUB_ORG}'" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

#### 4\. Allow Authentications from the Workload Identity Pool to your Service Account

Grant your service account the permission to be impersonated by identities from the Workload Identity Pool.

```bash
# TODO: Replace ${PROJECT_ID} with your GCP Project ID.
# TODO: Replace ${SERVICE_ACCT_NAME} with your service account name (e.g., "github-actions-sa").
# TODO: Replace ${WORKLOAD_IDENTITY_POOL_ID} with the full pool ID (obtained in step 2).
# TODO: Replace ${REPO} with your full GitHub repository name (e.g., "your-org/your-repo").

gcloud iam service-accounts add-iam-policy-binding "${SERVICE_ACCT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${WORKLOAD_IDENTITY_POOL_ID}/attribute.repository/${REPO}"
```

#### 5\. Extract the Workload Identity Provider Resource Name

This is the exact value you'll use for the **`WIP_ID` GitHub Secret**.

```bash
# TODO: Replace ${PROJECT_ID} with your GCP Project ID.
gcloud iam workload-identity-pools providers describe "my-repo" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github" \
  --format="value(name)"
```

The output of this command will be the value for your `WIP_ID` secret.

#### 6\. Grant Necessary Permissions to the Service Account for BigQuery

Finally, grant your service account the appropriate permissions to perform operations in BigQuery. The following roles are typically required for creating/managing datasets and tables, and running jobs. Always adhere to the **principle of least privilege** and adjust these roles based on your specific needs.

```bash
# TODO: Replace ${PROJECT_ID} with your GCP Project ID.
# TODO: Replace ${SERVICE_ACCT_NAME} with your service account name (e.g., "github-actions-sa").

# This role allows the service account to run BigQuery jobs (e.g., queries, data loading).
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SERVICE_ACCT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/bigquery.jobUser" \
  --project="${PROJECT_ID}"

# This role allows the service account to create, update, and delete datasets and tables.
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --project="${PROJECT_ID}" \
  --role="roles/bigquery.dataEditor" \
  --member="serviceAccount:${SERVICE_ACCT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

# This role allows the service account to view data in BigQuery tables.
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --project="${PROJECT_ID}" \
  --role="roles/bigquery.dataViewer" \
  --member="serviceAccount:${SERVICE_ACCT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
```