## Introduction

![nuke-from-orbit](https://i.imgflip.com/1o9ejc.jpg)

Sometimes you need a little more boom, so let's rain fire from the clouds... it's the only way to be sure.

This section contains a framework for a kubernetes-based distributed locustio cluster. Provided is an example of how to run a "real browser" based test of a
looker dashboard.

This is heavily derived from the official GCP locust/kubernetes tutorial, which can be found
[here.](https://cloud.google.com/solutions/distributed-load-testing-using-gke)

The instructions below are for GCP, but this can be run on any Kubernetes cluster in any environment. At a high level
the steps are:

1. Build a Docker image of your locust tests
2. Spin up a Kubernetes cluster
3. Create the master and worker deployments, the service, and any related secrets

## A note on scaling

Running headless browsers is a CPU-bound process. For this reason, if you are trying to time dashboard load times with Selenium I strongly recommend using CPU-optimized 
machine types for your nodes. The example below uses the default N1 machine types for demonstration purposes. A reading of the Kubernetes deployment config files reveals 
that the worker pods request 1 core. A good rule of thumb is each worker can simulate 2 real browsers with 1 core, so if you wanted to simulate 20 browsers you will need 
approximately 10 cores in your cluster. (probably slightly more to handle some overhead - e.g. The master pod itself requires comparitively few resources, but still 
needs some) Attempting to run workers with less CPU will result in degraded dashboard loading performance, leading to incorrect test results, as well as risk of pod eviction.

One more note: For these tests, one browser does not equal one user - each browser can make a new dashboard request
every second or two, meaning one browser can simulate the traffic of several human users. 

## Before you begin

Open Cloud Shell to execute the commands listed in this tutorial.

Define environment variables for the project id, region and zone you want to use for this tutorial.

    $ PROJECT=$(gcloud config get-value project)
    $ REGION=us-central1
    $ ZONE=${REGION}-b
    $ CLUSTER=gke-load-test
    $ gcloud config set compute/region $REGION
    $ gcloud config set compute/zone $ZONE

**Note:** Following services should be enabled in your project:
Cloud Build
Kubernetes Engine
Cloud Storage

    $ gcloud services enable \
        cloudbuild.googleapis.com \
        compute.googleapis.com \
        container.googleapis.com \
        containeranalysis.googleapis.com \
        containerregistry.googleapis.com

## Setup

1. Create GKE cluster

        $ gcloud container clusters create $CLUSTER \
                --zone $ZONE \
                --scopes "https://www.googleapis.com/auth/cloud-platform" \
                --num-nodes "3" \
                --enable-autoscaling --min-nodes "3" \
                --max-nodes "10" \
                --addons HorizontalPodAutoscaling,HttpLoadBalancing

        $ gcloud container clusters get-credentials $CLUSTER \
        --zone $ZONE \
        --project $PROJECT

2. Clone tutorial repo in a local directory on your cloud shell environment

        $ git clone <this-repository>

3. Build docker image and store it in your project's container registry

        $ pushd gke-load-test
        $ gcloud builds submit --tag gcr.io/$PROJECT/locust-tasks:latest docker-image/.


4. Replace [PROJECT_ID] in locust-master-controller.yaml and locust-worker-controller.yaml with the deployed endpoint and project-id respectively.

        $ sed -i -e "s/\[PROJECT_ID\]/$PROJECT/g" kubernetes-config/locust-master-controller.yaml
        $ sed -i -e "s/\[PROJECT_ID\]/$PROJECT/g" kubernetes-config/locust-worker-controller.yaml


5. Create a kubernetes secret called `website-creds` that contains two entries - `username` and `password` - that tie to
   the looker instance you are logging into:

        $ kubectl create secret generic website-creds --from-literal=username='<your username>'
        --from-literal=password='<your password>'

6. Deploy Locust master and worker nodes:

        $ kubectl apply -f kubernetes-config/locust-master-controller.yaml
        $ kubectl apply -f kubernetes-config/locust-master-service.yaml
        $ kubectl apply -f kubernetes-config/locust-worker-controller.yaml

7. Get the external ip of Locust master service

        $ EXTERNAL_IP=$(kubectl get svc locust-master -o yaml | grep ip | awk -F":" '{print $NF}')

8. Starting load testing
The Locust master web interface enables you to execute the load testing tasks against the system under test, as shown in the following image. Access the url as http://$EXTERNAL_IP:8089.

To begin, specify the total number of users to simulate and a rate at which each user should be spawned. Next, click Start swarming to begin the simulation. To stop the simulation, click **Stop** and the test will terminate. The complete results can be downloaded into a spreadsheet.

9. [Optional] Scaling clients
Scaling up the number of simulated users will require an increase in the number of Locust worker pods. To increase the number of pods deployed by the deployment, Kubernetes offers the ability to resize deployments without redeploying them. For example, the following command scales the pool of Locust worker pods to 20:

        $ kubectl scale deployment/locust-worker --replicas=20

## Cleaning up

    $ gcloud container clusters delete $CLUSTER --zone $ZONE