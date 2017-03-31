# Developing microservices locally with Google Container Engine (Kubernetes)

The [guestbook](https://cloud.google.com/container-engine/docs/tutorials/guestbook) tutorial for Kubernetes shows how to get a simple PHP and Redis application running in Kubernetes, but doesn't explain how you might edit and test the code. We'll show you how to set up a fast, productive development environment for coding on Kubernetes.

## Microservices

Microservices are an increasingly popular design pattern for cloud applications. In a microservices architecture, an individual application is broken into many small services that can be independently developed, tested, and released. While this approach has numerous benefits, microservices can also bring additional complexity into your overall architecture and workflow.

One of the areas of complexity is setting up a productive development environment for microservices. In a traditional web application, a development environment may consist of a database and the actual web application. In a microservices cloud application, an individual service may depend on multiple other services. Moreover, the service may also utilize cloud resources such as Amazon RDS or Google Cloud Pub/Sub. Setting up and maintaining a development environment with multiple services and cloud resources can be a lot of work. While there are [multiple approaches to setting up a development environment for microservices](https://www.datawire.io/guide/deployment/development-environments-microservices/), this tutorial will walk through setting up local development environment for microservices with your services running on a remote Kubernetes cluster.

## Technologies used

* [Kubernetes](https://kubernetes.io)
* [Google Container Engine](https://cloud.google.com/container-engine/)
* [Docker](https://www.docker.com)
* [Telepresence](https://www.datawire.io/telepresence/)
* [PHP](http://www.php.net/) and [Redis](https://redis.io/)

## Prerequisites and setup

In order to use this demo, you're going to need:

* A local system running either Linux or Mac OS X
* Access to a Kubernetes cluster (this tutorial will walk through setting up a cluster using Google Container Engine)

### Setting up your local laptop

To set up your laptop, you'll need to install a few basic components.

First, you'll want to install Docker. If you don't have Docker installed, follow the install instructions at https://www.docker.com/community-edition.

Next, we're going to want to install the `gcloud` and `kubectl` commands. Follow the instructions at https://cloud.google.com/sdk/downloads to download and install the Cloud SDK. Then, insure `kubectl` is installed:

```
% sudo gcloud components update kubectl
```

We need to install Telepresence, which will proxy your locally running service to GKE.

```
% curl -L https://github.com/datawire/telepresence/raw/0.22/cli/telepresence -o telepresence
% chmod +x telepresence
```

Move telepresence to somewhere on your $PATH, e.g.,:

```
mv telepresence /usr/local/bin
```

We'll also need to configure a local development environment for PHP. The Guestbook application is fairly simple, but it does depend on the Predis library. We'll need to install the [PEAR package manager](https://pear.php.net/manual/en/installation.getting.php), and then install the Predis library.

```
% curl -O https://pear.php.net/go-pear.phar
% php go-pear.par
% pear channel-discover pear.nrk.io   # You may need to add pear to your path
% pear install nrk/Predis
```

Finally, this tutorial uses a number of Kubernetes configuration files. To save some typing, clone the [telepresence GitHub](https://github.com/datawire/telepresence/) repository:

```
% git clone https://github.com/datawire/telepresence.git
```

All example files are in the [`examples/guestbook`](https://github.com/datawire/telepresence/tree/master/examples/guestbook) directory.

### Setting up Kubernetes in Google Container Engine

Setting up a production-ready Kubernetes cluster can be fairly complex, so we're going to use Google Container Engine in our example. If you already have a Kubernetes cluster handy, you can skip this section.

To set up a Kubernetes cluster in GKE, go to https://console.cloud.google.com, choose the Google Container Engine option from the menu, and then Create a Cluster.

The following gcloud command will create a small 2 node cluster in the us-central1-a region:

```
% gcloud container --project "PROJECT" clusters create "EXAMPLE_NAME" --zone "us-central1-a" --machine-type "n1-standard-1" --image-type "GCI" --disk-size "100" --scopes "https://www.googleapis.com/auth/compute","https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "2" --network "default" --enable-cloud-logging --enable-cloud-monitoring
```

## Installing the Guestbook application

The [Guestbook](https://cloud.google.com/container-engine/docs/tutorials/guestbook) sample application is a simple PHP application backed by a Redis database. We're going to set up Redis in the cloud, and run the PHP application locally on our laptop.

To get started, we need to authenticate to our cluster:

```
% gcloud container clusters get-credentials CLUSTER_NAME
% gcloud auth application-default login
```

### Setting up Redis

Now, let's install Redis in our Kubernetes cluster. We'll need to set up the Redis master deployment ([config](https://github.com/datawire/telepresence/blob/master/examples/guestbook/redis-master-deployment.yaml)), the Redis master service ([config](https://github.com/datawire/telepresence/blob/master/examples/guestbook/redis-master-service.yaml)), the Redis slave deployment ([config](https://github.com/datawire/telepresence/blob/master/examples/guestbook/redis-slave-deployment.yaml)), and the Redis slave service ([config](https://github.com/datawire/telepresence/blob/master/examples/guestbook/redis-slave-service.yaml)). If you don't want to download each of these files manually, these files are in the [`examples/guestbook`](https://github.com/datawire/telepresence/tree/master/examples/guestbook) directory of the Telepresence repository.

```
% kubectl create -f redis-master-deployment.yaml
% kubectl create -f redis-master-service.yaml
% kubectl create -f redis-slave-deployment.yaml
% kubectl create -f redis-slave-service.yaml
```

You can verify that everything is running:

```
% kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
redis-master-343230949-lpw91   1/1       Running   0          1d
redis-slave-132015689-dpp46    1/1       Running   0          1d
redis-slave-132015689-v06md    1/1       Running   0          1d
```

### Setting up the Guestbook frontend

We're going to run the Guestbook PHP frontend locally, so the first step is to download the Docker image.

```
% docker pull gcr.io/google_samples/gb-frontend:v4
```

You can quickly run the frontend service:

```
% docker run -it --publish=8080:80 gcr.io/google_samples/gb-frontend:v4
```

The `--publish` option maps the container's port 80 to the host's port 8080. Visit `localhost:8080` in your browser, and you'll see the Guestbook application. You won't be able to add a Guestbook entry, because we haven't connected the PHP application to Redis yet. Exit your Docker process by typing `Ctrl-C`.

### Connecting the Guestbook frontend to Redis

We're now going to use [Telepresence](https://datawire.github.io/telepresence) to create a virtual network between your local machine and the remote Kubernetes cluster. This way, the PHP application will be able to talk to remote cloud resources, and vice versa.

We'll start by deploying the Telepresence proxy onto the Kubernetes cluster using the [`telepresence-deployment.yaml`](https://github.com/datawire/telepresence/blob/master/examples/guestbook/telepresence-deployment.yaml) deployment file.

```
% kubectl create -f telepresence-deployment.yaml
```

Next, we need to deploy an externally visible load balancer. The [`frontend-service.yaml`](https://github.com/datawire/telepresence/blob/master/examples/guestbook/frontend-service.yaml) uses a selector for `app:guestbook` and `tier:backend`. We've cleverly used these labels in the `telepresence-deployment.yaml`, so the load balancer will talk directly to our proxy.

```
% kubectl create -f frontend-service.yaml
```

Now, we're going to start the local Telepresence client, which will spawn a special shell which connects to the proxy in the remote Kubernetes cluster.

```
% telepresence --deployment telepresence-deployment --expose 8080 --run-shell
```

In this special shell, change to the `examples/guestbook` directory, and start the frontend application. We'll need to know the directory where Predis is installed. You can figure this out by typing:

```
% pear config-get php_dir
```

Now, in the `examples/guestbook` directory, start PHP:

```
php -d include_path="PATH_TO_PEAR_DIR" -S 0.0.0.0:8080
```

It's time to check out our app in the browser. Let's look up the IP address of our external load balancer:

```
% kubectl get services
NAME           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
frontend       10.7.252.209   104.196.217.24   80:30563/TCP   25m
redis-master   10.7.248.117   <none>           6379/TCP       5d
redis-slave    10.7.245.58    <none>           6379/TCP       5d
```

Go to the external IP address of your load balancer (in the above example, 104.196.217.24). You should see the Guestbook application running. Typing into the submit box will show how your message is persisting to the Redis cluster.

### Editing your code

Now, open `index.html` from your shell and try renaming the Submit button to Go. Hit reload, and you'll immediately see your changes reflected live.

### Behind the scenes

What's going on behind the scenes? Your incoming request goes to the load balancer. The load balancer, as mentioned above, looks for the Telepresence proxy based on the `app:guestbook` and `tier:backend` labels. The proxy, which is running in the cloud Kubernetes environment, then sends those requests to the local Telepresence client, which passes the request to the PHP application.

## Additional Resources

* [Setting up a Python development environment for Docker](http://matthewminer.com/2015/01/25/docker-dev-environment-for-web-app.html) covers how to configure your Docker image for hot reload
* [Doing the same for NodeJS](http://fostertheweb.com/2016/02/nodemon-inside-docker-container/)
* The [Microservices Architecture Guide](https://www.datawire.io/guide) covers design patterns and HOWTOs in setting up an end-to-end microservices infrastructure
* The [Kubernetes tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/) gives a good walk-through of using Kubernetes, or visit the [Google Container Engine Quickstart](https://cloud.google.com/container-engine/docs/quickstart)

## Conclusion

Microservices, or service-oriented development, is a paradigm that is here to stay for cloud applications. The fledgling nature of microservices means that the tooling around developing, testing, and deploying microservices is still immature. Hopefully this tutorial shows a practical way to set up fast, local development of a microservice while being able to utilize cloud-resources running in a Kubernetes environment.