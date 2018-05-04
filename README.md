## Distributed Load Testing Using Kubernetes

This tutorial demonstrates how to conduct distributed load testing using [Kubernetes](http://kubernetes.io) and includes a sample web application, Docker image, and Kubernetes controllers/services. For more background refer to the [Distributed Load Testing Using Kubernetes](http://cloud.google.com/solutions/distributed-load-testing-using-kubernetes) solution paper.

## Prerequisites

* Google Cloud Platform account
* Install and setup [Google Cloud SDK](https://cloud.google.com/sdk/)

**Note:** when installing the Google Cloud SDK you will need to enable the following additional components:

* `App Engine Command Line Interface`
* `App Engine SDK for Python and PHP`
* `Compute Engine Command Line Interface`
* `gcloud app Python Extensions`
* `kubectl`

Before continuing, you can also set your preferred zone and project:

    $ gcloud config set compute/zone ZONE
    $ gcloud config set project PROJECT-ID

## Deploy Web Application

The `sample-webapp` folder contains a simple Google App Engine Python application as the "system under test". To deploy the application to your project use the `gcloud app deploy` command.

    $ gcloud app deploy sample-webapp/app.yaml --project=PROJECT-ID

**Note:** you will need the URL of the deployed sample web application when deploying the `locust-master` and `locust-worker` controllers.

## Deploy Controllers and Services

Before deploying the `locust-master` and `locust-worker` controllers, update each to point to the location of your deployed sample web application. Set the `TARGET_HOST` environment variable found in the `spec.template.spec.containers.env` field to your sample web application URL.

    - name: TARGET_HOST
      value: http://PROJECT-ID.appspot.com

### Update Controller Docker Image (Optional)

The `locust-master` and `locust-worker` controllers are set to use the pre-built `locust-tasks` Docker image, which has been uploaded to the [Google Container Registry](http://gcr.io) and is available at `gcr.io/cloud-solutions-images/locust-tasks`. If you are interested in making changes and publishing a new Docker image, refer to the following steps.

First, [install Docker](https://docs.docker.com/installation/#installation) on your platform. Once Docker is installed and you've made changes to the `Dockerfile`, you can build, tag, and upload the image using the following steps:

    $ gcloud container builds submit --tag gcr.io/PROJECT-ID/locust-tasks:latest .

**Note:** you are not required to use the Google Container Registry. If you'd like to publish your images to the [Docker Hub](https://hub.docker.com) please refer to the steps in [Working with Docker Hub](https://docs.docker.com/userguide/dockerrepos/).

Once the Docker image has been rebuilt and uploaded to the registry you will need to edit the controllers with your new image location. Specifically, the `spec.template.spec.containers.image` field in each controller controls which Docker image to use.

If you uploaded your Docker image to the Google Container Registry:

    image: gcr.io/PROJECT-ID/locust-tasks:latest

If you uploaded your Docker image to the Docker Hub:

    image: USERNAME/locust-tasks:latest

**Note:** the image location includes the `latest` tag so that the image is pulled down every time a new Pod is launched. To use a Kubernetes-cached copy of the image, remove `:latest` from the image location.

### Deploy Kubernetes Cluster

First create the [Google Kubernetes Engine](http://cloud.google.com/kubernetes-engine) cluster using the `gcloud` command as shown below. 

    $ gcloud container clusters create CLUSTER-NAME

### Deploy locust-master

Now that `kubectl` is setup, deploy the `locust-master-controller`:

    $ kubectl create -f locust-master-controller.yaml

To confirm that the Replication Controller and Pod are created, run the following:

    $ kubectl get rc
    $ kubectl get pods -l name=locust,role=master

Next, deploy the `locust-master-service`:

    $ kubectl create -f locust-master-service.yaml

This step will expose the Pod with an internal DNS name (`locust-master`) and ports `8089`, `5557`, and `5558`. As part of this step, the `type: LoadBalancer` directive in `locust-master-service.yaml` will tell Google Kubernetes Engine to create a Google Compute Engine forwarding-rule from a publicly avaialble IP address to the `locust-master` Pod. To view the newly created forwarding-rule, execute the following:

    $ kubectl get svc locust-master

### Deploy locust-worker

Now deploy `locust-worker-controller`:

    $ kubectl create -f locust-worker-controller.yaml

The `locust-worker-controller` is set to deploy 10 `locust-worker` Pods, to confirm they were deployed run the following:

    $ kubectl get pods -l name=locust,role=worker

To scale the number of `locust-worker` Pods, issue a replication controller `scale` command.

    $ kubectl scale --replicas=20 replicationcontrollers locust-worker

To confirm that the Pods have launched and are ready, get the list of `locust-worker` Pods:

    $ kubectl get pods -l name=locust,role=worker

**Note:** depending on the desired number of `locust-worker` Pods, the Kubernetes cluster may need to be launched with more than 3 compute engine nodes and may also need a machine type more powerful than n1-standard-1. Refer to the [`gcloud alpha container clusters create`](https://cloud.google.com/sdk/gcloud/reference/alpha/container/clusters/create) documentation for more information.

## Execute Tests

To execute the Locust tests, navigate to the IP address of your service (see above) and port `8089` and enter the number of clients to spawn and the client hatch rate then start the simulation.

## Deployment Cleanup

To teardown the workload simulation cluster, use the following steps. First, delete the Kubernetes cluster:

    $ gcloud container clusters delete CLUSTER-NAME

To delete the sample web application, visit the [Google Cloud Console](https://console.developers.google.com).

## License

This code is Apache 2.0 licensed and more information can be found in `LICENSE`. For information on licenses for third party software and libraries, refer to the `docker-image/licenses` directory.
