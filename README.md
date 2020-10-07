# CI/CD pipeline example
## Work in Progress
The idea of this project is to create a simple GitHub + Cloud Build + Docker + GKE pipeline with a Vue.js web app. WiP.
```
npm install -g @vue/cli
vue create my-app-vue

docker build -t my-app .
docker run -it -p 8080:8080 --rm --name my-app-c my-app

gcloud container clusters create gke-my-vue-app --zone=europe-west1-d --num-nodes=1 --machine-type=n1-standard-1

kubectl create -f deployment.yaml
kubectl create -f service.yaml
```
