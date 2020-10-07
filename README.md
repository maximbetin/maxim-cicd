# CI/CD pipeline example
## Work in Progress
The idea of this project is to create a simple GitHub + Cloud Build + Docker + GKE pipeline with a containerized Vue.js web app.

This is very much a Work in Progress, I will thoroughly detail the process and refine the pipeline in the coming weeks.

```
npm install -g @vue/cli
vue create my-app-vue

gcloud services enable containerregistry.googleapis.com

gcloud services enable container.googleapis.com 

gcloud services enable cloudbuild.googleapis.com

gcloud projects add-iam-policy-binding <PROJECT-ID> --member='serviceAccount:<CLOUD-BUILD-SA>' --role='roles/container.admin'

gcloud container clusters create gke-my-app-vue --zone=europe-west1-d --machine-type=n1-standard-2 --enable-autoscaling --min-nodes=1 --max-nodes=3

```
