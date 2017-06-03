# Deploying JupyterHub on gcloud

These are my notes on trying to set up JupyterHub on gcloud.  Basically, this is me following along on the awesome [zero to jupyterhub](http://zero-to-jupyterhub.readthedocs.io/en/latest/) article.


## Install `kubectl` plugin

If you don't have it already, install `kubectl`:

```
gcloud components install kubectl
```


## Create the kubernetes cluster

http://zero-to-jupyterhub.readthedocs.io/en/latest/create-k8s-cluster.html

```
gcloud container clusters create notebook-test \
    --num-nodes=3 \
    --machine-type=n1-highmem-2 \
    --zone=us-central1-b
```

You can see the [list of machine types](https://cloud.google.com/compute/docs/machine-types) or run this command:

```
gcloud compute machine-types list
```

Once the command completes, you can confirm it exists like this:

```
$ kubectl get node
NAME                                           STATUS    AGE       VERSION
gke-notebook-test-default-pool-fc9a005b-64z2   Ready     41s       v1.6.4
gke-notebook-test-default-pool-fc9a005b-rjhm   Ready     43s       v1.6.4
gke-notebook-test-default-pool-fc9a005b-tj84   Ready     44s       v1.6.4
```

## Set up Helm

[Helm](https://helm.sh/) is the kubernetes package manager.

```
$ brew install kubernetes-helm
==> Downloading https://homebrew.bintray.com/bottles/kubernetes-helm-2.4.2.sierr
######################################################################## 100.0%
==> Pouring kubernetes-helm-2.4.2.sierra.bottle.tar.gz
==> Using the sandbox
==> Caveats
Bash completion has been installed to:
  /usr/local/etc/bash_completion.d
==> Summary
🍺  /usr/local/Cellar/kubernetes-helm/2.4.2: 48 files, 122.4MB
```

Then you have to run `helm init`; this has to be done once per cluster.

```
$ helm init
Creating /Users/odewahn/.helm
Creating /Users/odewahn/.helm/repository
Creating /Users/odewahn/.helm/repository/cache
Creating /Users/odewahn/.helm/repository/local
Creating /Users/odewahn/.helm/plugins
Creating /Users/odewahn/.helm/starters
Creating /Users/odewahn/.helm/repository/repositories.yaml
$HELM_HOME has been configured at /Users/odewahn/.helm.

Tiller (the helm server side component) has been installed into your Kubernetes Cluster.
Happy Helming!
```


## Prepare an initial JupyterHub config file

This is from http://zero-to-jupyterhub.readthedocs.io/en/latest/setup-jupyterhub.html#setup-jupyterhub

Generate two random keys that you'll using the in the config file:

```
$ openssl rand -hex 32
b85e9538b93761f2336025a3d5696cc237ee26c8115979d90b86b08b0c326957
$ openssl rand -hex 32
f13056563eafb75ab062020dadef8c941b18f69da623e8af58554c06c585881a
```
Then create a file called `config.yaml` with the following contents:

```
hub:
  cookieSecret: "b85e9538b93761f2336025a3d5696cc237ee26c8115979d90b86b08b0c326957"
token:
  proxy: "f13056563eafb75ab062020dadef8c941b18f69da623e8af58554c06c585881a"
```

## Install JupyterHub with Helm

Use `helm install` to get it up and running:

```
$ helm install https://github.com/jupyterhub/helm-chart/releases/download/v0.3.1/jupyterhub-v0.3.1.tgz \
>     --name=jupyterhub-test \
>     --namespace=jupyterhub-test \
>     -f config.yaml
NAME:   jupyterhub-test
LAST DEPLOYED: Fri Jun  2 10:29:44 2017
NAMESPACE: jupyterhub-test
STATUS: DEPLOYED

RESOURCES:
==> v1/PersistentVolumeClaim
NAME        STATUS   VOLUME                       CAPACITY  ACCESSMODES  STORAGECLASS  AGE
hub-db-dir  Pending  hub-storage-jupyterhub-test  1s

==> v1/Service
NAME          CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
hub           10.11.246.154  <none>       8081/TCP      1s
proxy-api     10.11.240.251  <none>       8001/TCP      1s
proxy-public  10.11.254.221  <pending>    80:30746/TCP  1s

==> v1/Secret
NAME        TYPE    DATA  AGE
hub-secret  Opaque  2     1s

==> v1/ConfigMap
NAME          DATA  AGE
hub-config-1  14    1s

==> v1beta1/Deployment
NAME              DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
hub-deployment    1        1        1           0          1s
proxy-deployment  1        1        1           0          1s

==> v1beta1/StorageClass
NAME                                 TYPE
single-user-storage-jupyterhub-test  kubernetes.io/gce-pd  
hub-storage-jupyterhub-test          kubernetes.io/gce-pd  


NOTES:
Thank you for installing JupyterHub!

Your release is named jupyterhub-test and installed into the namespace jupyterhub-test.

You can find if the hub and proxy is ready by doing:

 kubectl --namespace=jupyterhub-test get pod

and watching for both those pods to be in status 'Ready'.

You can find the public IP of the JupyterHub by doing:

 kubectl --namespace=jupyterhub-test get svc proxy-public

It might take a few minutes for it to appear!

Note that this is still an alpha release! If you have questions, feel free to
  1. Come chat with us at https://gitter.im/jupyterhub/jupyterhub
  2. File issues at https://github.com/jupyterhub/helm-chart/issues
```

Once it starts, you can see where the cluster is running:

```
$ kubectl --namespace=jupyterhub-test get svc proxy-public
NAME           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
proxy-public   10.11.254.221   <pending>     80:30746/TCP   36s
```

Wait a minute or two until the external IP is available:

```
$ kubectl --namespace=jupyterhub-test get svc proxy-public
NAME           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
proxy-public   10.11.254.221   104.155.179.31   80:30746/TCP   4m
```

Then you can open you browser to `http://104.155.179.31`. JupyterHub is running with a default dummy authenticator, so you can just enter any username and password. (See [extending jupyterhub](http://zero-to-jupyterhub.readthedocs.io/en/latest/extending-jupyterhub.html) for details on how to set up authentication. )

<img src="images/jupyterhub-login.png"/>

## Prepare a Notebook for JupyterHub

To make a Docker image you can deploy onto JupyterHub, you need to `ADD` the repo to the `/home/jovyan` directory, and then set the `WORKDIR` to `/home/jovyan`.  

If you're using Launchbot or otherwise have an existing Dockerfile, you can create a new Dockerfile and call it `Dockerfile.prod`.  For example:

```
FROM jupyter/scipy-notebook:latest

ADD . /home/jovyan

WORKDIR /home/jovyan

# Expose the notebook port
EXPOSE 8888

# Start the notebook server
CMD jupyter notebook --no-browser --port 8888 --ip=*
```

Once you have this, build the image:

```
docker build -f Dockerfile.prod . aodewahn/thinkdsp
```

Then you'll need to log into the Docker Hub and create an image for it.  Once you do, you can do

```
docker push aodewahn/thinkdsp
```

<img width="100%" src="images/dockerhub.png"/>

## Deploying the Notebook on the cluster

The [extending jupyterhub](http://zero-to-jupyterhub.readthedocs.io/en/latest/extending-jupyterhub.html) article has about everything you can want.  It's pretty great.

For example, to specify a base image, add this to the `config.yaml` file:

```
singleuser:
  storage:
    type: none
  image:
    name: aodewahn/thinkdsp
    tag: latest
```

Next, get the release name of the app, which you set up earlier.  Note that if you forget the name, you can use `helm` to retrieve it:

```
$ helm list
NAME           	REVISION	UPDATED                 	STATUS  	CHART            	NAMESPACE      
jupyterhub-test	1       	Fri Jun  2 10:29:44 2017	DEPLOYED	jupyterhub-v0.3.1	jupyterhub-test
```

Then upgrade the cluster (this is what the doc says is necessary):

```
helm upgrade jupyterhub-test https://github.com/jupyterhub/helm-chart/releases/download/v0.3/jupyterhub-v0.3.tgz -f config.yaml
```

You can run this command until the container is built:

```
$  kubectl --namespace=jupyterhub-test get pod
NAME                                READY     STATUS              RESTARTS   AGE
hub-deployment-1010241260-x506q     0/1       ContainerCreating   0          57s
proxy-deployment-2951954964-l94n5   1/1       Running             0          2h
```

Once it's done building, you should be able to create a new notebook based on the base image.

<img width="100%" src="images/notebook.png"/>

Note that for now JupyterHub doesn't support persistent storage with the jupyter-stack images, but they're working on it.

## Delete the cluster

Once you're done, delete your cluster in order to stop further billing!

```
gcloud container clusters delete notebook-test --zone=us-central1-b
```
