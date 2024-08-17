---
title: "How to push a Docker image directly to Minikube"
author: "Connor Boyle"
---

The other day, I was attempting to run a service on my Knative set up, which was itself running on top of a Minikube
cluster. I assumed (incorrectly) that I could build a Docker image on the host machine and it would be automatically
available to Minikube. However, this is not true, because Minikube has its own Docker daemon, inside of its own virtual
machine (which, if your set-up is like mine, is itself running in a container on top of the host's Docker daemon). While
there *is* an easy and simple method that allows building a Docker image and pushing directly to your Minikube
cluster's Docker daemon, I don't believe it is well-documented anywhere on the public web, so I thought I would write my
own walkthrough.

The following walkthrough assumes that you have a running Minikube cluster and have installed `kubectl`.

### Un-installing Snap Docker

First, if you are on Ubuntu, you need to make sure that you are *not* running
the [Snap](https://ubuntu.com/core/services/guide/snaps-intro) version of Docker; the Docker client on your host machine
will need to authenticate to the Docker daemon on the Minikube host using a cert file that is inaccessible to the
Snap version of Docker, due to that Snap's containment policy. So make sure you have
installed [Docker Desktop](https://docs.docker.com/desktop/install/linux-install/) from the downloadable `.deb` file, or
add Docker's package repository
and [install Docker CE](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) using Apt.

## Connecting to the Minikube's Docker daemon

In order to connect to the Docker daemon inside the Minikube VM, we will need to change the values of certain
environment variables. Luckily, Minikube makes it easy for us to get these values with the following command:

```shell
minikube docker-env
```

(you will need to instead run `minikube -p <PROFILE-NAME> docker-env` if you want to connect to a Minikube profile other
than the currently activated one)

this should return an output similar to the following:

```
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.58.2:2376"
export DOCKER_CERT_PATH="/home/your-username/.minikube/certs"
export MINIKUBE_ACTIVE_DOCKERD="profile-name"

# To point your shell to minikube's docker-daemon, run:
# eval $(minikube -p profile-name docker-env)
```

Export these values to your current terminal's environment by running the command described in the last line of the
output, i.e.:

```commandline
eval $(minikube -p profile-name docker-env)
```

(note: `profile-name` will likely be a different value when run on your machine, you should copy & run the output of
*your* `minikube docker-env` command, not the one on this webpage)

You can verify that your Docker client has successfully connected to the Minikube VM Docker daemon by running:


```shell
$ docker images
REPOSITORY                                         TAG                                        IMAGE ID       CREATED         SIZE
registry.k8s.io/kube-apiserver                     v1.30.0                                    c42f13656d0b   4 months ago    117MB
registry.k8s.io/kube-controller-manager            v1.30.0                                    c7aad43836fa   4 months ago    111MB
registry.k8s.io/kube-scheduler                     v1.30.0                                    259c8277fcbb   4 months ago    62MB
registry.k8s.io/kube-proxy                         v1.30.0                                    a0bf559e280c   4 months ago    84.7MB
...
```

your output should similarly contain several images from the Kubernetes official registry.

**NOTE: the above will have to be re-run every time you open a new terminal, open a new SSH session, restart the computer,
etc.**

## Running the Image on Minikube

To test that we can actually build an image to the Minikube VM's Docker daemon, let's start by making a directory named
`test-docker`, then make a `Dockerfile` in it with the following contents:

```Dockerfile
FROM python
CMD python -c "print('Hello, world. This is Python, inside a Docker container, possibly on a Kubernetes cluster')"
```

In a terminal that has connected to the Minikube VM's Docker daemon (by following the instructions above), `cd` into the
parent directory of `test-docker`, then run:

```commandline
docker build --tag my-python test-docker/
```

Now check that the image is available by running:

```commandline
$ docker images
REPOSITORY                                         TAG                                        IMAGE ID       CREATED         SIZE
my-python                                          latest                                     17f99b663100   10 days ago     1.02GB
registry.k8s.io/kube-apiserver                     v1.30.0                                    c42f13656d0b   4 months ago    117MB
registry.k8s.io/kube-scheduler                     v1.30.0                                    259c8277fcbb   4 months ago    62MB
registry.k8s.io/kube-controller-manager            v1.30.0                                    c7aad43836fa   4 months ago    111MB
...
```

Now run a pod using this image with the following command:

```shell
$ kubectl run --image my-python --image-pull-policy Never my-python-pod
pod/my-python-pod created
```

(the [`--image-pull-policy Never`](https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy) is
necessary because Kubernetes looks for images in a default registry, without even considering images in its own
Docker daemon)

Check that the pod has run:

```shell
$ kubectl get pods
NAME                                             READY   STATUS                   RESTARTS         AGE
...
my-python-pod                                    0/1     Completed                2 (13s ago)      15s
```

(the pod's `STATUS` may eventually change to `CrashLoopBackOff`; I think this is because Kubernetes does not expect pods
to execute one command and then terminate)

You can see that the pod has completed the command described in the Dockerfile's CMD directive by running:

```shell
$ kubectl logs my-python-pod
Hello, world. This is Python, inside a Docker container, possibly on a Kubernetes cluster
```

And there you have it! A Docker image built and run on your local Minikube cluster.
