## Deploy on Google Kubernetes Engine (GKE)

### Before you begin

1. Set config for `gcloud`:
    ```sh
    export PROJECT_ID=<YOUR_PROJECT_ID>
    ```

    ```sh
    gcloud config set deploy/region us-central1
    gcloud config set artifacts/location us-central1
    gcloud config set project $PROJECT_ID
    ```

1. Enable APIs: 

    ```sh
    gcloud services enable 
        artifactregistry.googleapis.com \
        cloudbuild.googleapis.com \
        clouddeploy.googleapis.com \
        binaryauthorization.googleapis.com \
        container.googleapis.com \
        containeranalysis.googleapis.com \
        containerscanning.googleapis.com \
        containersecurity.googleapis.com
    ```

1. Download the [Cloud Code Source Protect Plugin]()

1. Make sure the default Compute Engine service account and Cloud Build service account have sufficient permissions.

    Note: The service account might already have the necessary permissions. These steps are included for projects that disable automatic role grants for default service accounts.

    * Grant the Cloud Build service account privilege:
        * to invoke deployments with Google Cloud Deploy and to update the delivery pipeline and the target definitions
        * to invoke Google Cloud Deploy operations (act as a service account)
        * to deploy to GKE

        ```sh
        gcloud projects add-iam-policy-binding $PROJECT_ID \
            --member=serviceAccount:$(gcloud projects describe $PROJECT_ID \
            --format="value(projectNumber)")@cloudbuild.gserviceaccount.com \
            --role="roles/clouddeploy.operator"

        gcloud projects add-iam-policy-binding $PROJECT_ID \
            --member=serviceAccount:$(gcloud projects describe $PROJECT_ID \
            --format="value(projectNumber)")@cloudbuild.gserviceaccount.com \
            --role="roles/iam.serviceAccountUser"

        gcloud projects add-iam-policy-binding $PROJECT_ID \
            --member=serviceAccount:$(gcloud projects describe $PROJECT_ID \
            --format="value(projectNumber)")@cloudbuild.gserviceaccount.com \
            --role="roles/container.admin"

        ```

    * Grant the Cloud Build and Google Cloud Deploy service account, default Compute Engine service account, privilege to deploy to GKE:

        ```sh
        gcloud projects add-iam-policy-binding $PROJECT_ID \
            --member=serviceAccount:$(gcloud projects describe $PROJECT_ID \
            --format="value(projectNumber)")-compute@developer.gserviceaccount.com \
            --role="roles/run.developer"

        gcloud projects add-iam-policy-binding $PROJECT_ID \
            --member=serviceAccount:$(gcloud projects describe $PROJECT_ID \
            --format="value(projectNumber)")-compute@developer.gserviceaccount.com \
            --role="roles/clouddeploy.jobRunner"
        ```

1. Replace PROJECT_ID placeholder with your Project Id:
    * MacOS
        ```sh
        sed -i '.bak' 's/PROJECT_ID/$PROJECT_ID/g' **/*clouddeploy.yaml policy.yaml pom.xml
        ```
    * Linux
        ```sh
        sed -i 's/PROJECT_ID/$PROJECT_ID/g' **/*clouddeploy.yaml policy.yaml pom.xml
        ```

### Setup

1. Set a Binary Authorization Policy:

    **⚠️ Note:** For your project to have the built by Cloud Build attestor, you need to run a build first. The easiest way to achieve this is deploying the backend to Cloud Run via source deploy: `cd backend; gcloud run deploy`

    ```sh
    gcloud container binauthz policy import policy.yaml
    ```

1. Create an Artifact Registry Docker repository:

    ```sh
    gcloud artifacts repositories create containers \
        --repository-format=docker \
        --description="Docker repository"
    ```

1. Create an Artifact Registry Maven repository:

    ```sh
    gcloud artifacts repositories create guestbook-maven-repo \
        --repository-format=maven \
        --location=us-central1 \
        --description="My Maven repo" 
    ```

1. Create an Artifact Registry remote repository ([learn more about remote repos](https://cloud.google.com/artifact-registry/docs/repositories/remote-repo)):

    ```sh
    gcloud artifacts repositories create guestbook-remote-repo \
        --repository-format=maven \
        --location=us-central1 \
        --description="My remote repo" \
        --mode=remote-repository \
        --remote-repo-config-desc="Maven Central" \
        --remote-mvn-repo=MAVEN-CENTRAL
    ```

1. Create clusters with Binary Authorization enabled:

    ```sh
    gcloud container clusters create-auto dev-cluster --region=us-central1 --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE && \
    gcloud container clusters create-auto prod-cluster --region=us-central1 --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE
    ```

1. Create your Cloud Deploy delivery pipeline and targets:

    ```sh
    gcloud deploy apply --file clouddeploy.yaml
    ```
   **⚠️ Note:** Ensure clouddeploy.yaml has the correct values. “PROJECT_ID” should have been already replaced with your Project Id


### Demo

1. Use the Cloud Code Source Protect Plugin to view dependencies vulnerabilities:

    In [`backend/pom.xml`](./backend/pom.xml) locate the `com.google.code.gson:gson` dependency. The plugin should advise to update the dependency to v2.8.9+.

1. Submit the Cloud Build:

    ```sh
    gcloud builds submit --config cloudbuild.yaml --substitutions SHORT_SHA=1234
    ```
    The build does the following:
    * Caches dependency artifacts into an Artifact Registry Maven Packages remote repo. The first time that you request a version of a package, Artifact Registry downloads and caches the package in the remote repository. The next time you request the same package version, Artifact Registry serves the cached copy.
    * Builds and stores a Java dependency artifacts to Artifact Registry with provenance
    * Builds and push containers to Artifact Registry
    * Automatically signs the artifacts with the attestor: “built-with-cloud-build”
    * Creates a release via Cloud Deploy

1. View the container vulnerabilities, dependencies, and provenance via Cloud Build:
    * Open the [Cloud Build console](https://console.cloud.google.com/cloud-build/builds)
    * Click on the build ID to view the build
    * Click the "Build Artifacts" tab
    * Click "View" under "Security Insights"
    * Click on "Artifacts scanned" to view vulnerabilities in Artifact Registry

1. View the Maven artifact provenance via Cloud Build:
    * TODO

1. View artifacts cached in the Artifact Registry remote repository:

    ```sh
    gcloud artifacts files list --repository=guestbook-remote-repo
    ```

1. View **GKE Security Postures**:
    * [Navigate to the GKE security posture management UI](https://console.cloud.google.com/kubernetes/security/dashboard)
    * View the cluster concerns and vulnerabilities

1. Deploy the release to production via Cloud Deploy:
    * [Navigate to the Cloud Deploy pipeline](https://console.cloud.google.com/deploy/delivery-pipelines/us-central1/cloudrun-guestbook-backend-delivery)
    * Click "Promote"

### Test the Binary Authorization policy

1. Build the container locally via Jib (Cloud Build signs builds with built-by-cloudbuild attestor, so this will not be signed):

    ```sh
    cd backend
    mvn compile jib:build \
    -Dimage=us-central1-docker.pkg.dev/$PROJECT_ID/containers/java-guestbook-backend:blocked

    cd frontend
    mvn compile jib:build \
    -Dimage=us-central1-docker.pkg.dev/$PROJECT_ID/containers/java-guestbook-frontend:blocked

    ```

1. Try to deploy the images:

    ```sh
    export _FRONTEND_IMAGE=us-central1-docker.pkg.dev/$PROJECT_ID/containers/java-guestbook-backend:blocked
    export _BACKEND_IMAGE=us-central1-docker.pkg.dev/$PROJECT_ID/containers/java-guestbook-backend:blocked

    gcloud deploy releases create release-blocked \
    --region us-central1 \
    --delivery-pipeline=guestbook-app-delivery \
    --images=java-guestbook-frontend=${_FRONTEND_IMAGE},java-guestbook-backend=${_BACKEND_IMAGE}
    ```