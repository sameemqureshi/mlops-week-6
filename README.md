# Iris Classifier API on GKE with GitHub Actions CD
This project sets up a Continuous Deployment (CD) pipeline for a simple Iris flower classification API. The API is built using FastAPI, containerized with Docker, and deployed to a Google Kubernetes Engine (GKE) cluster using GitHub Actions.

Project Overview
The core components of this project are:

Iris Classifier API: A Python-based API built with FastAPI that loads a pre-trained scikit-learn model (model.joblib) to predict Iris flower species based on input features.

Docker: Used to containerize the FastAPI application, making it portable and runnable across different environments.

Kubernetes (K8s): Provides an orchestration platform for deploying, managing, and scaling the containerized API on GKE.

Google Kubernetes Engine (GKE): Google Cloud's managed Kubernetes service.

Google Artifact Registry: Stores the Docker images securely.

GitHub Actions: Automates the build, push, and deployment process whenever changes are pushed to the main branch of the GitHub repository.

Project Structure
```
├── .github/
│   └── workflows/
│       └── cd.yaml              # GitHub Actions workflow for Continuous Deployment
├── docker_version/
│   ├── Dockerfile               # Defines the Docker image for the Iris API
│   ├── iris_fastapi.py          # FastAPI application code
│   ├── model.joblib             # Pre-trained scikit-learn model
│   ├── requirements.txt         # Python dependencies for the API
│   └── kubernetes/
│       ├── deployment.yaml      # Kubernetes Deployment manifest
│       └── service.yaml         # Kubernetes Service manifest (LoadBalancer)
└── README.md                    # This README file
```
GCP Setup
1. Created a GKE Cluster
```
gcloud container clusters create iris-cluster \
  --num-nodes=3 \
  --disk-size=20GB \
  --zone=us-central1-a
```

2. Created an Artifact Registry Repository
```
gcloud artifacts repositories create iris-api \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker repo for ML models"
```

3. Created a GCP Service Account and Grant Roles
This service account will be used by GitHub Actions to interact with GCP.

Artifact Registry Writer: Allows pushing images to Artifact Registry.

Kubernetes Engine Developer: Allows deploying applications to GKE.

Service Account User: Allows the service account to impersonate other service accounts (e.g., GKE nodes' service account for kubectl access).

Downloaded the Service Account Key (JSON): On the Service Accounts page, click the three dots next to github-actions-sa, select Manage keys, then ADD KEY > Create new key > JSON. Download the JSON file. Keep this file secure and do NOT commit it to Git.

4. Added GitHub Secrets
Storing  GCP credentials securely as GitHub Secrets:

Click New repository secret.

Name: GCP_PROJECT_ID

Secret: Your Google Cloud Project ID (e.g., thinking-return-461215-i0).

Click New repository secret again.

Name: GCP_SA_KEY

Secret: The entire JSON content from the service account key file you downloaded. Ensure no extra spaces or newlines.

Continuous Deployment
The cd.yaml file defines the CD pipeline. Ensure its content matches the final version provided below.

.github/workflows/cd.yaml (Final Version):

```
name: Continuous Deployment to GKE

on:
  push:
    branches:
      - main # Trigger this workflow on pushes to the 'main' branch

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  GKE_CLUSTER: iris-cluster
  GKE_ZONE: us-central1-a
  DEPLOYMENT_NAME: iris-api-deployment
  SERVICE_NAME: iris-api-service
  
  # Define parts for the image path.
  IMAGE_HOST: us-central1-docker.pkg.dev 
  ARTIFACT_REGISTRY_REPO_NAME: iris-api 
  APP_IMAGE_NAME: iris-classifier-api # The name of your Docker image artifact
  IMAGE_TAG: latest

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          project_id: ${{ env.PROJECT_ID }}
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Google Cloud SDK and Install GKE Auth Plugin
        uses: google-github-actions/setup-gcloud@v2
        with:
          project_id: ${{ env.PROJECT_ID }}
          install_components: 'gke-gcloud-auth-plugin' # Install the necessary kubectl auth plugin

      - name: Authenticate Docker to Artifact Registry
        run: gcloud auth configure-docker ${{ env.IMAGE_HOST }}

      - name: Build Docker image
        run: docker build -t ${{ env.IMAGE_HOST }}/${{ env.PROJECT_ID }}/${{ env.ARTIFACT_REGISTRY_REPO_NAME }}/${{ env.APP_IMAGE_NAME }}:${{ env.IMAGE_TAG }} .

      - name: Push Docker image to Artifact Registry
        run: docker push ${{ env.IMAGE_HOST }}/${{ env.PROJECT_ID }}/${{ env.ARTIFACT_REGISTRY_REPO_NAME }}/${{ env.APP_IMAGE_NAME }}:${{ env.IMAGE_TAG }}

      - name: Get GKE credentials
        run: gcloud container clusters get-credentials ${{ env.GKE_CLUSTER }} --zone ${{ env.GKE_ZONE }}

      - name: Deploy to GKE
        run: |
          FULL_IMAGE_PATH="${{ env.IMAGE_HOST }}/${{ env.PROJECT_ID }}/${{ env.ARTIFACT_REGISTRY_REPO_NAME }}/${{ env.APP_IMAGE_NAME }}:${{ env.IMAGE_TAG }}"
          
          sed -i "s|gcr.io/\[YOUR_GCP_PROJECT_ID\]/iris-api:latest|${FULL_IMAGE_PATH}|g" kubernetes/deployment.yaml

          kubectl apply -f kubernetes/deployment.yaml
          kubectl apply -f kubernetes/service.yaml
          echo "Waiting for deployment rollout to complete..."
          kubectl rollout status deployment/${{ env.DEPLOYMENT_NAME }}

          echo "Service details:"
          kubectl get service ${{ env.SERVICE_NAME }}

Testing the API
Once the GitHub Actions workflow completes successfully:

Send a POST Request:
Once you have the EXTERNAL-IP (e.g., 34.123.45.67), use curl to send a POST request to your API's /predict/ endpoint:

curl -X POST \
  http://YOUR_EXTERNAL_IP/predict/ \
  -H 'Content-Type: application/json' \
  -d '{
    "sepal_length": 5.1,
    "sepal_width": 3.5,
    "petal_length": 1.4,
    "petal_width": 0.2
  }'

we should receive a JSON response, e.g.:

{"predicted_class":"versicolor"}

