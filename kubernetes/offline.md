date: 2016-05-05
author: Ivan Pedrazas
tags: baremetal, offline
status: draft

Kubernetes Offline
==========================

Lately I've had to create a Kubernetes installation that it's completely offline. This is, the cluster has no internet access whatsoever. I know, I know, you might think, why do you need internet access anyway, but you see, you do... !

A typical production architecture can be found [here](/kubernetes/production.html), but these are the components:

* ETCD cluster
* Docker Registry
* Kubernetes Master
* Worker Nodes

Kubernetes uses a `pause` image (` gcr.io/google_containers/pause:2.0`), in order to use it
