---
title: "It's weirdly hard to push a Docker image to Minikube"
author: "Connor Boyle"
---

The other day, I was attempting to run a service on my Knative set up, which was itself running on top of a Minikube
cluster. I assumed (incorrectly) that I could build a Docker image on the host machine and it would be automatically
available to Minikube. However, this is not possible (without a bit of additional work) because Minikube has its own
Docker daemon, inside of its own virtual machine (which, if you're like me is itself running in a container on top of
the host's Docker daemon).
