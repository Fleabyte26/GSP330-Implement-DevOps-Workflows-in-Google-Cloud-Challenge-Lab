# GSP-Implement-DevOps-Workflows-in-Google-Cloud-Challenge-Lab

README.md
Markdown
# Google Cloud Challenge Lab: GSP330 Reference Guide
### Implement DevOps Workflows in Google Cloud

> **Note for Students:** Your lab assigns an individual `REGION` and `ZONE`. Update the variables in **Step 1** before running any commands.
Step 1: Set Your Assigned Variables
Check your lab instructions table under Task 1, enter your assigned REGION and ZONE, then run this entire block in Cloud Shell:

Bash
# ==============================================================================
# SET YOUR ASSIGNED REGION AND ZONE
# ==============================================================================
export REGION=""    # e.g. us-central1
export ZONE=""        # e.g. us-central1-a

# Automated context variables
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=\((gcloud projects describe\)PROJECT_ID --format='value(projectNumber)')
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE

export GIT_SERVER_IP=\((gcloud compute instances describe git-server --zone=\)ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
echo "Git Server IP is: ${GIT_SERVER_IP}"
Step 2: Task 1 — Create Lab Resources
Enable required APIs, grant IAM permissions, configure Git, create the Artifact Registry repository, deploy the GKE cluster, and configure namespaces:

Bash
# Enable APIs
gcloud services enable container.googleapis.com cloudbuild.googleapis.com

# Grant Cloud Build Service Account Kubernetes Developer role
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
    --role="roles/container.developer"

# Configure Git
git config --global user.name "Student"
git config --global user.email "student@qwiklabs.net"

# Create Artifact Registry Docker Repository
gcloud artifacts repositories create my-repository \
    --repository-format=docker \
    --location=$REGION \
    --description="Docker repository for sample app"

# Create GKE Standard cluster
gcloud container clusters create hello-cluster \
    --zone=$ZONE \
    --release-channel=regular \
    --enable-autoscaling \
    --num-nodes=3 \
    --min-nodes=2 \
    --max-nodes=6

# Get cluster credentials and create namespaces
gcloud container clusters get-credentials hello-cluster --zone=$ZONE
kubectl create namespace prod
kubectl create namespace dev
Check Progress: Click Check my progress on "Create the lab resources".

Step 3: Task 2 — Connect to Git Server & Push Sample Code
Download the starter code, replace placeholders, and commit to both the master and dev branches:

Bash
cd ~
gcloud storage cp -r gs://spls/gsp330/sample-app/* sample-app

# Replace placeholders with project region/zone and set v1.0
for file in sample-app/cloudbuild-dev.yaml sample-app/cloudbuild.yaml; do
    sed -i "s//\({REGION}/g" "\)file"
    sed -i "s//\({ZONE}/g" "\)file"
    sed -i "s//v1.0/g" "$file"
done

# Initialize git, commit, and push to master
cd ~/sample-app
git init
git remote add origin http://${GIT_SERVER_IP}:3000/giteaadmin/sample-app.git
git branch -m master
git add .
git commit -m "initial commit"
git push -u http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/sample-app.git master

# Create and push dev branch
git checkout -b dev
git push -u http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/sample-app.git dev
Step 4: Task 3 — Create Cloud Build Triggers
Create the manual triggers for production and development deployments:

Bash
# Create Prod Trigger
gcloud builds triggers create manual \
    --name="sample-app-prod-deploy" \
    --inline-config="cloudbuild.yaml" \
    --service-account="projects/\({PROJECT_ID}/serviceAccounts/\){PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
    --region=$REGION

# Create Dev Trigger
gcloud builds triggers create manual \
    --name="sample-app-dev-deploy" \
    --inline-config="cloudbuild-dev.yaml" \
    --service-account="projects/\({PROJECT_ID}/serviceAccounts/\){PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
    --region=$REGION
Check Progress: Click Check my progress on "Create the Cloud Build Triggers".

Step 5: Task 4 — Deploy First Versions (v1.0)
1. Dev Deployment (v1.0)
Bash
cd ~/sample-app
git checkout dev

# Replace  container image placeholder in dev/deployment.yaml
sed -i "s||\({REGION}-docker.pkg.dev/\){PROJECT_ID}/my-repository/dev:v1.0|g" dev/deployment.yaml

git add .
git commit -m "Deploy v1.0 on dev"
git push http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/sample-app.git dev

# Run build
gcloud builds submit --config=cloudbuild-dev.yaml .

# Expose dev deployment
kubectl expose deployment development-deployment -n dev \
    --name=dev-deployment-service \
    --type=LoadBalancer \
    --port=8080 \
    --target-port=8080
2. Prod Deployment (v1.0)
Bash
cd ~/sample-app
git checkout master

# Replace  container image placeholder in prod/deployment.yaml
sed -i "s||\({REGION}-docker.pkg.dev/\){PROJECT_ID}/my-repository/prod:v1.0|g" prod/deployment.yaml

git add .
git commit -m "Deploy v1.0 on master"
git push http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/sample-app.git master

# Run build
gcloud builds submit --config=cloudbuild.yaml .

# Expose prod deployment
kubectl expose deployment production-deployment -n prod \
    --name=prod-deployment-service \
    --type=LoadBalancer \
    --port=8080 \
    --target-port=8080
Check Progress: Click Check my progress on "Deploy the first versions of the application".

Step 6: Task 5 — Deploy Second Versions (v2.0)
1. Update Application Code
Bash
cd ~/sample-app

# Overwrite main.go with /red route and redHandler support
cat <<'EOF' > main.go
package main

import (
	"image"
	"image/color"
	"image/draw"
	"image/png"
	"net/http"
)

func main() {
	http.HandleFunc("/blue", blueHandler)
	http.HandleFunc("/red", redHandler)
	http.ListenAndServe(":8080", nil)
}

func blueHandler(w http.ResponseWriter, r *http.Request) {
	img := image.NewRGBA(image.Rect(0, 0, 100, 100))
	draw.Draw(img, img.Bounds(), &image.Uniform{color.RGBA{0, 0, 255, 255}}, image.ZP, draw.Src)
	w.Header().Set("Content-Type", "image/png")
	png.Encode(w, img)
}

func redHandler(w http.ResponseWriter, r *http.Request) {
	img := image.NewRGBA(image.Rect(0, 0, 100, 100))
	draw.Draw(img, img.Bounds(), &image.Uniform{color.RGBA{255, 0, 0, 255}}, image.ZP, draw.Src)
	w.Header().Set("Content-Type", "image/png")
	png.Encode(w, img)
}
EOF
2. Dev Deployment (v2.0)
Bash
git checkout dev

# Update image version to v2.0
sed -i "s/v1.0/v2.0/g" cloudbuild-dev.yaml
sed -i "s/v1.0/v2.0/g" dev/deployment.yaml

git add .
git commit -m "Deploy v2.0 on dev"
git push http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/sample-app.git dev

# Run build
gcloud builds submit --config=cloudbuild-dev.yaml .
3. Prod Deployment (v2.0)
Bash
git checkout master

# Pull main.go changes into master
git checkout dev -- main.go

# Update image version to v2.0
sed -i "s/v1.0/v2.0/g" cloudbuild.yaml
sed -i "s/v1.0/v2.0/g" prod/deployment.yaml

git add .
git commit -m "Deploy v2.0 on master"
git push http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/sample-app.git master

# Run build
gcloud builds submit --config=cloudbuild.yaml .
Check Progress: Click Check my progress on "Deploy the second versions of the application".

Step 7: Task 6 — Roll Back Production Deployment
Roll back the deployment in the prod namespace directly to the previous revision (v1.0):

Bash
kubectl rollout undo deployment/production-deployment -n prod
Wait ~15 seconds, then verify the rollout completed:

Bash
kubectl rollout status deployment/production-deployment -n prod
Check Progress: Click Check my progress on "Roll back the production deployment". Your score will now be 100 / 100.
