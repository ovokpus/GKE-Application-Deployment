# Cymbal Superstore E-commerce CI/CD Pipeline

This playbook describes how to set up a complete CI/CD pipeline for a sample Go application using Google Kubernetes Engine (GKE), Artifact Registry, GitHub (Cloud Build GitHub App), and Cloud Build triggers. The pipeline has two environments: development (`dev`) and production (`prod`).

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Task 1: Create the Lab Resources](#task-1-create-the-lab-resources)
3. [Task 2: Create and Populate the GitHub Repository](#task-2-create-and-populate-the-github-repository)
4. [Task 3: Create Cloud Build Triggers](#task-3-create-cloud-build-triggers)
5. [Task 4: Deploy Version 1.0 of the Application](#task-4-deploy-version-10-of-the-application)
   1. [Development Environment (v1.0)](#development-environment-v10)
   2. [Production Environment (v1.0)](#production-environment-v10)
6. [Task 5: Deploy Version 2.0 of the Application](#task-5-deploy-version-20-of-the-application)
   1. [Development Environment (v2.0)](#development-environment-v20)
   2. [Production Environment (v2.0)](#production-environment-v20)
7. [Task 6: Roll Back the Production Deployment](#task-6-roll-back-the-production-deployment)
8. [Useful Commands](#useful-commands)

---

## Prerequisites

- A Google Cloud project with billing enabled.
- gcloud, kubectl, and GitHub CLI (gh) installed (Cloud Shell has these tools pre-installed).
- Sufficient IAM permissions to create and configure GKE, Artifact Registry, and Cloud Build triggers.

---

## Task 1: Create the Lab Resources

1. **Enable Required APIs**

   ```bash
   gcloud services enable container.googleapis.com \
       cloudbuild.googleapis.com \
       sourcerepo.googleapis.com
   ```

2. **Create an Artifact Registry Docker Repository**

   ```bash
   gcloud artifacts repositories create my-repository \
       --repository-format=docker \
       --location=us-east4 \
       --description="Docker repository for container images."
   ```

3. **Create a GKE Standard Cluster**

   ```bash
   gcloud container clusters create hello-cluster \
       --zone=us-east4-a \
       --release-channel=regular \
       --cluster-version=1.29 \
       --num-nodes=3 \
       --enable-autoscaling \
       --min-nodes=2 \
       --max-nodes=6
   ```

   > **Note:** Adjust cluster version as needed (e.g., `1.29.0-gke.x`) if required.

4. **Get Cluster Credentials**

   ```bash
   gcloud container clusters get-credentials hello-cluster --zone us-east4-a
   ```

5. **Create Namespaces**

   ```bash
   kubectl create namespace prod
   kubectl create namespace dev
   ```

6. **Grant the Kubernetes Developer Role to Cloud Build**

   ```bash
   export PROJECT_ID=$(gcloud config get-value project)
   gcloud projects add-iam-policy-binding $PROJECT_ID \
       --member="serviceAccount:$(gcloud projects describe $PROJECT_ID \
       --format="value(projectNumber)")@cloudbuild.gserviceaccount.com" \
       --role="roles/container.developer"
   ```

---

## Task 2: Create and Populate the GitHub Repository

1. **Create a GitHub Repository**

   ```bash
   gh repo create sample-app --public --confirm
   ```

2. **Clone the Repository**

   ```bash
   gh repo clone sample-app
   cd sample-app
   ```

3. **Copy the Sample Code**

   ```bash
   cd ~
   gsutil cp -r gs://spls/gsp330/sample-app/* sample-app
   ```

   Return to your repository folder:

   ```bash
   cd ~/sample-app
   ```

4. **Commit and Push Initial Code**

   ```bash
   git add .
   git commit -m "Initial commit of sample-app"
   git push -u origin master
   ```

---

## Task 3: Create Cloud Build Triggers

1. **sample-app-prod-deploy (Production)**

   - **Name:** `sample-app-prod-deploy`
   - **Event:** Push to a branch
   - **Branch Regex:** `^master$`
   - **Build Configuration File:** `cloudbuild.yaml`

2. **sample-app-dev-deploy (Development)**
   - **Name:** `sample-app-dev-deploy`
   - **Event:** Push to a branch
   - **Branch Regex:** `^dev$`
   - **Build Configuration File:** `cloudbuild-dev.yaml`

These triggers will build and deploy to the `prod` or `dev` namespaces whenever changes are pushed to `master` or `dev` branches, respectively.

---

## Task 4: Deploy Version 1.0 of the Application

### Development Environment (v1.0)

1. **Switch to dev Branch**

   ```bash
   git checkout -b dev
   ```

2. **Update `cloudbuild-dev.yaml`**

   - Change `<version>` to `v1.0` wherever needed.

3. **Update `dev/deployment.yaml`**

   - Replace `<todo>` with your container image name, including the v1.0 tag, for example:  
     `us-east4-docker.pkg.dev/PROJECT_ID/my-repository/sample-app:v1.0`

4. **Commit and Push**

   ```bash
   git add .
   git commit -m "Deploy dev v1.0"
   git push -u origin dev
   ```

   This triggers the **sample-app-dev-deploy** pipeline.

5. **Verify Build in Cloud Build**

   - Check **Cloud Build** > **History** and confirm the build succeeded.

6. **Expose the Deployment**

   ```bash
   kubectl expose deployment development-deployment \
       --type=LoadBalancer \
       --name=dev-deployment-service \
       --port=8080 --target-port=8080 \
       -n dev
   ```

   - After an external IP is assigned, test it:  
     `http://<EXTERNAL_IP>:8080/blue`

### Production Environment (v1.0)

1. **Switch to master Branch**

   ```bash
   git checkout master
   ```

2. **Update `cloudbuild.yaml`**

   - Change `<version>` to `v1.0` wherever needed.

3. **Update `prod/deployment.yaml`**

   - Replace `<todo>` with the v1.0 image:
     `us-east4-docker.pkg.dev/PROJECT_ID/my-repository/sample-app:v1.0`

4. **Commit and Push**

   ```bash
   git add .
   git commit -m "Deploy prod v1.0"
   git push origin master
   ```

   This triggers the **sample-app-prod-deploy** pipeline.

5. **Verify Build in Cloud Build**

   - Check **Cloud Build** > **History** for the build’s status.

6. **Expose the Deployment**

   ```bash
   kubectl expose deployment production-deployment \
       --type=LoadBalancer \
       --name=prod-deployment-service \
       --port=8080 --target-port=8080 \
       -n prod
   ```

   - Test it:  
     `http://<EXTERNAL_IP>:8080/blue`

---

## Task 5: Deploy Version 2.0 of the Application

### Development Environment (v2.0)

1. **Switch to dev Branch**

   ```bash
   git checkout dev
   ```

2. **Update Code**  
   In `main.go`, add/update the following:

   ```go
   func main() {
       http.HandleFunc("/blue", blueHandler)
       http.HandleFunc("/red", redHandler)
       http.ListenAndServe(":8080", nil)
   }

   func redHandler(w http.ResponseWriter, r *http.Request) {
       img := image.NewRGBA(image.Rect(0, 0, 100, 100))
       draw.Draw(img, img.Bounds(), &image.Uniform{color.RGBA{255, 0, 0, 255}}, image.ZP, draw.Src)
       w.Header().Set("Content-Type", "image/png")
       png.Encode(w, img)
   }
   ```

3. **Update `cloudbuild-dev.yaml`**

   - Change `<version>` or the existing version reference to `v2.0`.

4. **Update `dev/deployment.yaml`**

   - Change the container image to `v2.0`.

5. **Commit and Push**

   ```bash
   git add .
   git commit -m "Deploy dev v2.0 with /red endpoint"
   git push origin dev
   ```

   The **sample-app-dev-deploy** pipeline builds and deploys again.

6. **Verify the Deployment**
   - Check Cloud Build History and confirm success.
   - Verify the `/red` endpoint:  
     `http://<EXTERNAL_IP>:8080/red`

### Production Environment (v2.0)

1. **Switch to master Branch**

   ```bash
   git checkout master
   ```

2. **Update Code** (the same changes for `/red`):

   ```go
   func main() {
       http.HandleFunc("/blue", blueHandler)
       http.HandleFunc("/red", redHandler)
       http.ListenAndServe(":8080", nil)
   }

   func redHandler(w http.ResponseWriter, r *http.Request) {
       // ...
   }
   ```

3. **Update `cloudbuild.yaml`**

   - Change the image version to `v2.0`.

4. **Update `prod/deployment.yaml`**

   - Change the container image to `v2.0`.

5. **Commit and Push**

   ```bash
   git add .
   git commit -m "Deploy prod v2.0 with /red endpoint"
   git push origin master
   ```

   The **sample-app-prod-deploy** pipeline builds and deploys again.

6. **Verify the Deployment**
   - Check Cloud Build History to ensure success.
   - Verify the `/red` endpoint:
     `http://<EXTERNAL_IP>:8080/red`

---

## Task 6: Roll Back the Production Deployment

If you need to revert to the previous version (v1.0) for production:

1. **Roll Back Using kubectl**

   ```bash
   kubectl rollout undo deployment/production-deployment -n prod
   ```

2. **Confirm Deployment**

   ```bash
   kubectl rollout status deployment/production-deployment -n prod
   ```

3. **Test the `/red` Endpoint**
   - Since v1.0 doesn’t include `/red`, navigating to `http://<EXTERNAL_IP>:8080/red` should return a 404 error.

---

## Useful Commands

- **View Deployments in a Namespace**

  ```bash
  kubectl get deployments -n <namespace>
  ```

- **View Services in a Namespace**

  ```bash
  kubectl get services -n <namespace>
  ```

- **Get External IP of a Service**

  ```bash
  kubectl get service <service-name> -n <namespace>
  ```

- **View Rollout History**

  ```bash
  kubectl rollout history deployment/<deployment-name> -n <namespace>
  ```

- **Undo a Deployment Rollout**

  ```bash
  kubectl rollout undo deployment/<deployment-name> -n <namespace>
  ```

- **Set Image Pull Policy to Always**

  ```yaml
  containers:
    - name: sample-container
      image: us-east4-docker.pkg.dev/<PROJECT_ID>/<REPO>/<IMAGE>:v1.0
      imagePullPolicy: Always
  ```

---

**Congratulations!** You have now set up a fully functioning CI/CD pipeline for both development and production environments, complete with automated builds, deployments, versioning, and rollback capabilities.
