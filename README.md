# Continuous Integration/Continuous Delivery with GCP

## 1 Preface

I recently started indulging myself in the world of DevOps — something that has captivated my attention quite quickly. Although I’ll be honest, it took me a lot of reading and lots of YouTube binge watching to understand the concepts. Even to this day, I’m not confident enough to say that I’ve fully assimilated all of the principles, practices, values and ideals that coalesce into the concept of DevOps.

Regardless, I’ve decided to share my knowledge and illustrate one of the core tenets of DevOps — automation and CI/CD. With a simple demo involving some of the tools and knowledge I’ve acquired throughout my time in Cloud and Software Development, I hope I will be able to help in paving the way to foster this knowledge for myself and others.

## 2 Introduction

This is a simple demo on the usage of Google's [Cloud Build](https://cloud.google.com/cloud-build), [Kubernetes Engine](https://cloud.google.com/kubernetes-engine) and [Container Registry](https://cloud.google.com/container-registry), with [Docker](https://www.docker.com/), [Vue.js](https://vuejs.org/), [Git](https://git-scm.com/) and [GitHub](https://github.com/) to build a CI/CD pipeline. Here’s a diagram that summarizes the workflow:

<p align="center">
  <img src="https://snipboard.io/ef8aE6.jpg">
</p>

1. A developer pushes the source code to GitHub, which lends itself as the Version Control System.
2. GitHub triggers a post-commit hook to Cloud Build.
3. Cloud Build creates the container image and pushes it to Container Registry.
4. Cloud Build notifies Google Kubernetes Engine to roll out a new deployment.
5. Google Kubernetes Engine pulls the image from Container Registry and runs it.

CI/CD will allow us to shorten that feedback loop and cycle time. Instead of delivering code in large batches after many months, it is delivered in small iterations over the existing code, taking as little as days or hours. As a result, improving rhythm of practice, productivity and security. It empowers teams to push code more efficiently and effectively, helping bring value, which benefits the organization as a whole by building high quality, robust and reliable systems.

## 3 Defining the baseline and CI/CD requirements

Before we begin, some obvious things to note are that you will need to replace many of the values I use to define my project’s specifications with your own, these especially are the `[PROJECT_ID]`, `[PROJECT_NUMBER]` which you will need to replace with your own Project ID and Project Number respectively. The name of the Google Kubernetes Engine cluster, Cloud Build trigger, Docker image name, etc. can be kept the same if you wish

I’ll also spare you the pain of having to look and poke around at a GUI and thus use the command-line interface as much as possible to achieve what we’re after — after all, I do advocate for avoiding unnecessary clicks and automation. The [`gcloud`](https://cloud.google.com/sdk/gcloud/reference) CLI will serve us well here.

Let’s start off by tackling the non-cloud related tools. Back in the day, I decided to build myself a portfolio website and after some research, I chose Vue.js as the framework and I wasn’t disappointed. I was quite impressed by how simple, user-friendly and quick to set up it was. A few years ago all I heard about was PHP, Javascript and .NET in web development. Therefore I’ll use Vue.js as the web/application framework. But enough small talk.

Open up a Terminal in your Linux distro and run these commands:

1. `sudo npm install -g @vue/cli` to install `vue`.
2. `sudo vue create my-app-vue` to create a preset website in Vue.js. You might get prompted to select a Vue version, I will use `Vue 2`.
3. Oh wait, that’s it. Done. Yes, you’ve got yourself a website template after running a grand total of two Linux commands. **Success**.

Side-note: if you come across a `npm ERR! code EINTEGRITY` error during the installation run `sudo npm cache verify`, it usually fixes it.

Moving on. Docker. What is it? Plainly put Docker carves up a computer into a sealed, portable container that runs your code. Think of it as the minimal amount of software and OS needed to run your code — a lightweight VM. And yes, I just angered and antagonized the entire Docker community against me after describing Docker that way.

The idea revolves around containerizing the code we just created into a Docker image. The image has your code and everything else that you deem necessary for it to be running, with no underlying hardware prerequisites.

Docker images are built using the `Dockerfile` where we define these requirements. The `Dockerfile` contains all the steps needed to run our particular piece of software — all the commands to assemble the image.

This is the `Dockerfile` we will be using:

```
# Get a base Node image
FROM node:alpine

# Make the 'app' folder the current working directory
WORKDIR /app

# Copy both 'package.json' and 'package-lock.json' (if present)
COPY package*.json ./

# Install project dependencies
RUN npm install

# Copy project files and folders to the current working directory (i.e. 'app' folder)
COPY . .

# Build app for production with minification
RUN npm run build

# Runs the website on http://localhost:8080
CMD ["npm", "run", "serve"]
```

Every `Dockerfile` command serves a function, I’m not going to explain how every single one of them works, that would make this demo excessively long (longer than it probably already is), I invite you to visit the Dockerfile reference [documentation](https://docs.docker.com/engine/reference/builder/) for that instead. But in a nutshell what we do is:

- Specify the base image — `node:alpine`.
- Copy all the dependencies and necessary files into the working directory.
- Build the application with `RUN npm run build`.
- Specify the instruction to execute when the Docker container starts with `CMD ["npm", "run", "serve"]`.

In order to test that application is built and the containerized web application functions correctly run:

1. `docker build -t gcr.io/[PROJECT_ID]/my-app-vue:latest .` to generate the image, tag it and upload it to Google Container Registry.
2. `docker run -p 80:8080 gcr.io/[PROJECT_ID]/my-app-vue:latest` to run a container on port `80`.
3. Navigate to http://localhost:80 in your browser and you should see the website running:

<p align="center">
  <img src="https://snipboard.io/9kMNdp.jpg">
</p>

4. Hit `Ctrl+C` in the terminal to stop the container.

An advantage of using a CI/CD pipeline is that we don’t have to do the 4 previous steps. The pipeline itself will build the image by running that exact command, upload it to GCR and update the GKE cluster with the same image automatically in the subsequent instructions.

On the other hand, The CI (Continuous Integration) part of the pipeline integrates the code, it performs the necessary static and dynamic tests and verifies the code integrity. In this way we can automate the deployment of new Docker images and abstract the process of deployment from the developer, whose main focus will be on code-related tasks. I’m going to omit running tests in the pipeline as that would imply me having to write them, which would be rather cumbersome to demonstrate a simple use-case of a CI/CD pipeline, but bear in mind that different types of tests are indeed run in it. Some examples are functional testing, unit testing, load testing, stress testing, integration testing and regression testing, perhaps being **unit, integration and acceptance testing**, the pivotal types of testing that you will often hear thrown around in DevOps. Integration tests are often (and counterintuitively) performed on the CD part of the pipeline, and conversely, unit tests in CI.

Last but not least, it’s important to note the value that the use of containers in DevOps provide — mainly consistency. The `Dockerfile` provides a consistent definition on how to create an application and the environment it needs to run. Meaning, that the images we create, can be used across development, testing and production environments, simulating the application’s behaviour where needed. And since a `Dockerfile`  is all but a text file, it can be easily tracked and shared in Version Control systems. The images themselves that are created can be also be tracked in Docker registries such as the public Docker Hub, but since GCP is the highlight of this demo, we will be using Google’s Container Registry instead as mentioned earlier.

## 4 Creating the Google Cloud Platform environment
### 4.1 Initial setup

Now here is where [Google Cloud Platform](https://cloud.google.com/) comes into play. All the Docker images that we upload will be stored in Google’s Container Registry and rolled out to a Google Kubernetes Engine cluster. But let’s not get ahead of ourselves, first, we need to set up a new GCP project for this demo. If you have a spare one you can use lying around, feel free to use it instead. These are the commands you need to run in order to create one:

1. `gcloud projects create [PROJECT-ID] --name="CI-CD Test"` creates the project.
2. `gcloud config set project [PROJECT-ID]` sets the local [Cloud SDK](https://cloud.google.com/sdk) configuration to point to this project.

Then link the newly created project to your billing account with `gcloud beta billing projects link [PROJECT-ID] --billing-account=0X0X0X-0X0X0X-0X0X0X`. Replace the value `0X0X0X-0X0X0X-0X0X0X` with your billing account's number.

Now, let’s go ahead and enable the APIs that we will be needing here:

- `gcloud services enable containerregistry.googleapis.com` to enable Google’s Container Registry API.
- `gcloud services enable container.googleapis.com` to enable Google’s Kubernetes Engine API. This one might take a while — patience.
- `gcloud services enable cloudbuild.googleapis.com` to enable the Cloud Build API.

At this point the project is ready. The previous `gcloud` commands are a one-time action per project, and we do not need to run them again, in fact you could even run a script for them if you plan on creating multiple projects. 

Frankly, if you’re going to create multiple projects that use multiple services that’s where [Terraform](https://www.terraform.io/) or Google’s [Deployment Manager](https://cloud.google.com/deployment-manager) would come in handy, but that’s a story for another time. 

### 4.2 Creating a Google Kubernetes Engine cluster

We can now get our hands dirtier and start by creating the services we will be using here.

Starting off with Google Kubernetes Engine — Google’s managed orchestration system for deploying, managing, and scaling your containerized applications in the Cloud. Kubernetes is a world of its own, and I could never do it justice by summing it up in a paragraph, trying to do so I might as well end up writing a thesis on it.

Concepts that are important to know here are [Pods](https://cloud.google.com/kubernetes-engine/docs/concepts/pod) — where Docker containers will run, [Deployments](https://cloud.google.com/kubernetes-engine/docs/concepts/deployment) — a declarative specification of the application that will run on the cluster and a [Service](https://cloud.google.com/kubernetes-engine/docs/concepts/service) which defines how these Pods and the application in whole will be accessed by the end user. If you’re new to K8S and GKE, I highly advise to have a look at the documentation.

GKE clusters are more often than not used in an auto scaling configuration, especially in CI/CD. I’m going to create an auto scaling cluster with up to 3 nodes in the `europe-west1-d` zone of the machine type `n1-standard-2`:

```
gcloud container clusters create gke-my-app-vue \
--zone=europe-west1-d \
--machine-type=n1-standard-2 \
--enable-autoscaling \
--min-nodes=1 \
--max-nodes=3
```

We will use the standard `YAML` file K8S uses for the Deployment and the Service. Both are in the `/k8s` directory of the repository. 

The Deployment specifies the port that the containers will be exposing and points to the image in Google Container Registry:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: my-app-vue
  name: my-app-vue
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-app-vue
  template:
    metadata:
      labels:
        run: my-app-vue
    spec:
      containers:
      - image: gcr.io/[PROJECT-ID]/my-app-vue:latest
        name: my-app-vue
        ports:
        - containerPort: 8080
```
Note that you have to replace [PROJECT-ID] with the value of your own Project ID.

The Service on the other hand, specifies the K8S Service type we will be using which is the the commonly used [`LoadBalancer`](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer), you can use any other Service type you wish:

```
kind: Service
apiVersion: v1
metadata:
  name: my-app-vue
spec:
  selector:
     run: my-app-vue
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
```
  
We could have actually defined both the Deployment and the Service in one single `YAML` file, isolating each section with [`---`](https://kubernetes.io/docs/contribute/style/style-guide/#paragraphs), but for the sake of the demo I will keep them separated.

We also could apply this configuration on the GKE cluster with [`kubectl apply -f`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply) at this point, but it's not necessary since the pipeline will do that for us anyway. Also the image hasn't been uploaded to GCR yet either, so the GKE cluster deployment would probably fail. Furtunately enough, the Cloud Build pipeline will be tasked with and solve all of that.

### 4.3 Setting up Cloud Build

#### 4.3.1 Writing the build steps

Once that’s done, we can move on to Cloud Build. Cloud Build is one of the essential services for building CI/CD pipelines within GCP. It is a continuous build, test, and deploy pipeline service managed by Google, similar to Bitbucket or GitLab pipelines. Cloud Build parses a file in `YAML` syntax called `cloudbuild.yaml`, analogous to the `bitbucket-pipelines.yml` file in Bitbucket or `gitlab-ci.yml` in GitLab, albeit with Google’s own distinctions and variations. Cloud Build offers 120 minutes of free build time in GCP and it’s well integrated with the rest of GCP’s products that we’re going to use so it will suit us well for the purpose of the demo.

The file will contain the steps necessary to create the build — creating the Docker image, pushing it to Google Container Registry and setting the GKE cluster to use it:

```
steps:
# Step 1
- name: 'gcr.io/cloud-builders/docker'
  entrypoint: 'bash'
  args: [
   '-c',
   'docker pull gcr.io/$PROJECT_ID/my-app-vue:latest || exit 0'
  ]
# Step 2
- name: gcr.io/cloud-builders/docker
  args: [
   'build',
   '-t',
   'gcr.io/$PROJECT_ID/my-app-vue:latest',
   '.'
  ]
  dir: 'my-app-vue'
# Step 3
- name: 'gcr.io/cloud-builders/kubectl'
  args: ['apply', '-f', 'k8s/']
  env:
  - 'CLOUDSDK_COMPUTE_ZONE=europe-west1-d'
  - 'CLOUDSDK_CONTAINER_CLUSTER=gke-my-app-vue'
# Step 4
- name: 'gcr.io/cloud-builders/kubectl'
  args: [
   'set',
   'image',
   'deployment',
   'my-app-vue',
   'my-app-vue=gcr.io/$PROJECT_ID/my-app-vue:latest'
  ]
  env:
  - 'CLOUDSDK_COMPUTE_ZONE=europe-west1-d'
  - 'CLOUDSDK_CONTAINER_CLUSTER=gke-my-app-vue'
  # Push the image to Google Container Registry with the latest tag
images: [
   'gcr.io/$PROJECT_ID/my-app-vue:latest'
  ]

````
- In Step 1, we are attempting to pull an existing latest Docker to leverage Docker’s cached layers functionality of old images to build new ones. This quite a nifty concept, I’ve seen images get reduced to a mere 4MB when they would otherwise take a few GBs. `|| exit 0` is used in case the `docker pull` command returns an error (when running this build for the first time for instance, there will be no latest image to pull from the repository), the entire pipeline would fail. 

- In Step 2, the Docker image is built. This is pretty straightforward and equivalent to running the `docker build -t gcr.io/[PROJECT_ID]/my-app-vue:latest .` command mentioned earlier.

- In Step 3 we apply the Kubernetes configuration on the cluster by parsing the `YAML` files. We’re essentially running `kubectl apply -f` on both of the files we created earlier. The environment variables specified after `env:` tell `gcloud` which GKE cluster and which zone we want to apply these changes to.

- In Step 4, all that’s left is to point the cluster’s deployment to the Docker image residing in GCR, and K8S will take care of the rest.

From the coding perspective, that’s it, we’re almost set to start triggering the pipeline. All that’s left is to give the appropriate permissions to [Cloud Build’s Service Account](https://cloud.google.com/cloud-build/docs/cloud-build-service-account). Why? Because Cloud Build will be the service pushing the changes to the GKE cluster, and it won’t be able to unless it has the permissions.

The Cloud Build Service Account normally has the format of `[PROJECT_NUMBER]@cloudbuild.gserviceaccount.com`, to find how it is displayed in your own project you can run `gcloud projects get-iam-policy` and `grep` the output:

`gcloud projects get-iam-policy [PROJECT-ID] | grep "cloudbuild.gserviceaccount.com"`

I’m going to give it the [`roles/container.admin`](https://cloud.google.com/kubernetes-engine/docs/how-to/iam#predefined) role in another `gcloud projects` command:

```
gcloud projects add-iam-policy-binding [PROJECT-ID] \
--member='serviceAccount:[PROJECT_NUMBER]@cloudbuild.gserviceaccount.com' \
--role='roles/container.admin'
```

Where `[PROJECT-ID]` is the ID of the Project and `[PROJECT_NUMBER]` is the Project Number. 

#### 4.3.2 Connecting GitHub to GCP

Now we are ready to connect the dots. We are going to add a Cloud Build trigger to initiate our pipeline. First of all, we need to connect this GitHub repository to Cloud Build.

Unfortunately life isn’t fair sometimes, and we can’t automate and script everything out through the CLI. We will have to go through the heinous act of clicking at a GUI since there is no `gcloud` command for this at the moment. Not to worry, I will apply my skills in taking screenshots for this part:

Open up the [Cloud Console](https://console.cloud.google.com/), and navigate to Cloud Build. Select `Trigger`s an click on `Connect Repository`:

<p align="center">
  <img src="https://snipboard.io/zV062N.jpg">
</p>

Select GitHub as the source:

<p align="center">
  <img src="https://i.ibb.co/PsLBk38/image.png">
</p>

Select the GitHub account, repository and click on `Connect Repository`:

<p align="center">
  <img src="https://i.ibb.co/mRFV6nk/image.png">
</p>

Now we will be presented with the option of creating a push trigger. I prefer to skip this step and create it with a `gcloud` command instead, but feel free to opt into creating it through the Cloud Console instead, I won’t hold it against you:

<p align="center">
  <img src="https://snipboard.io/qhYi0x.jpg">
</p>

If all steps were done correctly, we should at least see a connected repository banner under the Inactive tab now:

<p align="center">
  <img src="https://snipboard.io/6xnVc4.jpg">
</p>

#### 4.3.3 Creating the Cloud Build trigger

We’re almost there, all that’s left is creating the Cloud Build trigger. The [`gcloud builds`](https://cloud.google.com/sdk/gcloud/reference/beta/builds) command is currently in Beta, but that shouldn’t be an problem, I’ve used `gcloud beta` commands multiple times and I can’t recall I had a single issue. 

Let’s get back on track. With the trigger, we’re telling Cloud Build what to trigger on (duh). Since our intent is to trigger a build whenever a `git push` is run on the GitHub’s repository `main` branch (also coincidentally GitHub’s usual default `master` branch was [renamed](https://www.infoq.com/news/2020/10/github-main-branch/) to `main` a week prior to writing this), we will specify that with the flag `--branch-pattern` set to `^main$`, and specify `github` after `create`. 

We’re also going to indicate where the `cloudbuild.yaml` file is located with the flag `--build-config`, which happens to be in the `root` directory in this case. The flags `--repo-name` and `--repo-owner` have to be set to the name of the repository and the owner (me in this case). I’m also going to set a name for the trigger and a description with the flags `--name` and `--description` respectively. 

```
gcloud beta builds triggers create github \
--name=maxim-build-trigger \
--description="Push to branch" \
--repo-name=maxim-cicd \
--repo-owner=maximbetin \
--branch-pattern="^main$" \
--build-config=cloudbuild.yaml
```

And finally we’re ready. If all went well, the final product should look similar to this:

<p align="center">
  <img src="https://snipboard.io/YfRBSI.jpg">
</p>

## 5 Testing the configuration

Getting to the gist of the demo, we can now either go ahead and click that `Run trigger` button to test it out, or what’s more fun is to trigger it with a `git push` and see magic happen, since this is also, what we intended from the beggining.

We only really need to run `git commit -am “New build”` followed by a `git push`, which will subsequently create a Docker image as instructed in the `cloudbuild.yaml` file, push it to Google Container Registry, and update the GKE cluster with it. 

If the build ran successfully, navigating to the `History` tab we should see a green indicator that it has been completed:

![Build](https://snipboard.io/6O1yRY.jpg)

### 5.1 View the built image

To verify that the image has been built and uploaded to Google Container Registry, you can either navigate to GCR from the Cloud Console or run `gcloud container images list --repository=gcr.io/[PROJECT-ID]`. Either way, checking GCR the image should be visible:

![Images](https://snipboard.io/jVzAm6.jpg)

Further clicking on the image, we will see any other versions of it that have been uploaded along with additional information such as the tags or the creation dates.

### 5.2 Observe the results

Obtain the GKE cluster’s Service IP by running:

- `gcloud container clusters get-credentials gke-my-app-vue --zone=europe-west1-d` in order to obtain the credentials to access the cluster.
- `kubectl get svc/my-app-vue` and see the Exposed IP of the Service `my-app-vue`.

Using your browser of choice navigate to GKE cluster’s exposed IP on port `80` and if you see the following — Success!

<p align="center">
  <img src="https://snipboard.io/ALmJwC.jpg">
</p>

## 6 In retrospect

This was a very simple example of the use of Google Cloud Build, Google Container Registry and Google Kubernetes Engine to create a CI/CD pipeline.

I hope that I was helpful, provided some new insight and came a step closer to indoctrinating you into the practice of CI/CD and DevOps — some of the core principles that allows us to break that barrier between Development and Operations teams — beliefs that are still deeply ingrained in many organizations.

And with that being said, thank you for stopping by and having a look. This was one of the most detailed guides I’ve written (the writing took me longer than the coding) even though I’ve omitted a lot of information, and also my first Medium post! Don’t hesitate reaching me out if you have any doubts. You can find me out [LinkedIn](https://www.linkedin.com/in/maxim-betin/). Have a good day!


