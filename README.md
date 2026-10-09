# GSP330 – Implement DevOps Workflows in Google Cloud: Challenge Lab

Paste each block into Cloud Shell. Use the copy button.

## 1. Setup

Replace the region and zone with the values in your lab instructions.

```bash

echo us-east1 > ~/region
echo us-east1-b > ~/zone
gcloud config get-value project > ~/project
gcloud projects describe `cat ~/project` --format="value(projectNumber)" > ~/projnum
gcloud config set compute/region `cat ~/region`
gcloud config set compute/zone `cat ~/zone`
gcloud compute instances describe git-server --zone=`cat ~/zone` --format="get(networkInterfaces[0].accessConfigs[0].natIP)" > ~/gitip
cat ~/project ~/projnum ~/region ~/zone ~/gitip

```

## 2. Task 1: create the lab resources

Cluster takes 5–8 min.

```bash

gcloud services enable container.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com
gcloud projects add-iam-policy-binding `cat ~/project` --member=serviceAccount:`cat ~/projnum`@cloudbuild.gserviceaccount.com --role=roles/container.developer --condition=None --quiet
gcloud projects add-iam-policy-binding `cat ~/project` --member=serviceAccount:`cat ~/projnum`-compute@developer.gserviceaccount.com --role=roles/container.developer --condition=None --quiet
git config --global user.name "Student"
git config --global user.email "student@qwiklabs.net"
gcloud artifacts repositories create my-repository --repository-format=docker --location=`cat ~/region`
gcloud container clusters create hello-cluster --zone=`cat ~/zone` --release-channel=regular --enable-autoscaling --num-nodes=3 --min-nodes=2 --max-nodes=6
gcloud container clusters get-credentials hello-cluster --zone=`cat ~/zone`
kubectl create namespace prod
kubectl create namespace dev

```

Check my progress: Create the lab resources

## 3. Task 2: push the code to the Git server

```bash

cd ~
mkdir -p sample-app
gcloud storage cp -r gs://spls/gsp330/sample-app/* sample-app
cd ~/sample-app
sed -i "s/<your-region>/`cat ~/region`/g; s/<your-zone>/`cat ~/zone`/g; s/<version>/v1.0/g" cloudbuild-dev.yaml cloudbuild.yaml
grep -o "my-repository/[a-z0-9-]*" cloudbuild-dev.yaml | head -1 | cut -d/ -f2 > ~/devimg
grep -o "my-repository/[a-z0-9-]*" cloudbuild.yaml | head -1 | cut -d/ -f2 > ~/prodimg
cat ~/devimg ~/prodimg
git init
git remote add origin http://`cat ~/gitip`:3000/giteaadmin/sample-app.git
git branch -m master
git add . && git commit -m "initial commit"
git push -u http://giteaadmin:GiteaPassword123@`cat ~/gitip`:3000/giteaadmin/sample-app.git master
git checkout -b dev
git push -u http://giteaadmin:GiteaPassword123@`cat ~/gitip`:3000/giteaadmin/sample-app.git dev

```

## 4. Task 3: Cloud Build triggers

```bash

cd ~/sample-app
gcloud builds triggers create manual --name=sample-app-prod-deploy --inline-config=cloudbuild.yaml --service-account=projects/`cat ~/project`/serviceAccounts/`cat ~/projnum`-compute@developer.gserviceaccount.com --region=`cat ~/region`
gcloud builds triggers create manual --name=sample-app-dev-deploy --inline-config=cloudbuild-dev.yaml --service-account=projects/`cat ~/project`/serviceAccounts/`cat ~/projnum`-compute@developer.gserviceaccount.com --region=`cat ~/region`

```

Check my progress: Create the Cloud Build Triggers

## 5. Task 4: deploy v1.0 to dev

```bash

cd ~/sample-app
git checkout dev
sed -i "s#<todo>#`cat ~/region`-docker.pkg.dev/`cat ~/project`/my-repository/`cat ~/devimg`:v1.0#" dev/deployment.yaml
grep image: dev/deployment.yaml
git add . && git commit -m "Deploy v1.0 on dev"
git push http://giteaadmin:GiteaPassword123@`cat ~/gitip`:3000/giteaadmin/sample-app.git dev
gcloud builds submit --config=cloudbuild-dev.yaml .
kubectl -n dev expose deployment development-deployment --name=dev-deployment-service --type=LoadBalancer --port=8080 --target-port=8080
until kubectl -n dev get svc dev-deployment-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}' | grep -q .; do sleep 10; done
kubectl -n dev get svc dev-deployment-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}' > ~/devip
sleep 30; curl -I http://`cat ~/devip`:8080/blue

```

Expect 200 OK.

## 6. Task 4: deploy v1.0 to prod

```bash

cd ~/sample-app
git checkout master
sed -i "s#<todo>#`cat ~/region`-docker.pkg.dev/`cat ~/project`/my-repository/`cat ~/prodimg`:v1.0#" prod/deployment.yaml
grep image: prod/deployment.yaml
git add . && git commit -m "Deploy v1.0 on master"
git push http://giteaadmin:GiteaPassword123@`cat ~/gitip`:3000/giteaadmin/sample-app.git master
gcloud builds submit --config=cloudbuild.yaml .
kubectl -n prod expose deployment production-deployment --name=prod-deployment-service --type=LoadBalancer --port=8080 --target-port=8080
until kubectl -n prod get svc prod-deployment-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}' | grep -q .; do sleep 10; done
kubectl -n prod get svc prod-deployment-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}' > ~/prodip
sleep 30; curl -I http://`cat ~/prodip`:8080/blue

```

Expect 200 OK.

Check my progress: Deploy the first versions of the application

## 7. Task 5: v2.0 with the red handler (dev, then prod)

```bash

cat > ~/red.go <<'EOF'

func redHandler(w http.ResponseWriter, r *http.Request) {
	img := image.NewRGBA(image.Rect(0, 0, 100, 100))
	draw.Draw(img, img.Bounds(), &image.Uniform{color.RGBA{255, 0, 0, 255}}, image.ZP, draw.Src)
	w.Header().Set("Content-Type", "image/png")
	png.Encode(w, img)
}
EOF
cd ~/sample-app
git checkout dev
grep -q redHandler main.go || { sed -i '/HandleFunc("\/blue"/a\	http.HandleFunc("/red", redHandler)' main.go; cat ~/red.go >> main.go; }
sed -i "s/v1.0/v2.0/g" cloudbuild-dev.yaml dev/deployment.yaml
git add . && git commit -m "Deploy v2.0 on dev"
git push http://giteaadmin:GiteaPassword123@`cat ~/gitip`:3000/giteaadmin/sample-app.git dev
gcloud builds submit --config=cloudbuild-dev.yaml .
git checkout master
grep -q redHandler main.go || { sed -i '/HandleFunc("\/blue"/a\	http.HandleFunc("/red", redHandler)' main.go; cat ~/red.go >> main.go; }
sed -i "s/v1.0/v2.0/g" cloudbuild.yaml prod/deployment.yaml
git add . && git commit -m "Deploy v2.0 on master"
git push http://giteaadmin:GiteaPassword123@`cat ~/gitip`:3000/giteaadmin/sample-app.git master
gcloud builds submit --config=cloudbuild.yaml .
sleep 60; curl -I http://`cat ~/devip`:8080/red; curl -I http://`cat ~/prodip`:8080/red

```

Expect 200 OK on both.

Check my progress: Deploy the second versions of the application

## 8. Task 6: roll back prod to v1.0

```bash

kubectl -n prod set image deployment/production-deployment "*=`cat ~/region`-docker.pkg.dev/`cat ~/project`/my-repository/`cat ~/prodimg`:v1.0"
kubectl -n prod rollout status deployment/production-deployment
kubectl -n prod get deployment production-deployment -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
sleep 30; curl -I http://`cat ~/prodip`:8080/red

```

Expect 404 Not Found.

Check my progress: Roll back the production deployment
