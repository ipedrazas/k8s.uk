date: 2016-05-11
author: Ivan Pedrazas
tags: usecase, prod

Kubernetes in Production
==========================

One of the most asked questions I get is, by far, if Kubernetes is ready from prime time. The answer is 100% afirmative. Then a stream of different questions come through: from references from clients to very detailed technical questions about different aspects: How do you do HA? what kind of problems did you have while running production clusters? how many nodes do you run? do you run databases in your clusters? and the list goes and goes and goes.

Both, Alex and myself, will be answering these and other questions in this blog, but today I'm going to draft how to build a typical kubernetes cluster. It's not our intention, right now, to provide an universal installer, but if you are interested on having such a wonderful tool, please let us know!

A kubernetes cluster is defined by 3 components:

* ETCD: it's the state holder. Etcd is the place where we keep the state of the cluster.
* Master: It's in charge of scheduling, assigning and removing tasks.
* Nodes: it's in charge of executing the scheduled tasks.

When building a production cluster you have to make a few decissions. Sizing is the first thing you will have to decide. A minimal prod configuration would be the following:

* Etcd: (3/5) nodes
* Master: (2/3) nodes
* Nodes: (3/*) nodes

The next thing that you have to decide is where to host your cluster. It is not the same to host your cluster in Amazon AWS than in-house in your own datacenter.

Once you have defined the size of your cluster, it's time to create the cluster. I will not get into details of which CM to use, which one is good and which one is better. That's entirely up to you, but what I do can say it's that the more Kubernetes you use, the less CM you will need.

There's a pattern that we've been using successfully for a while and that is to run as much as possible in containers. Yes, this means running ETCD as containers also. Running containers is great, that's why we're moving towards Kubernetes, so, let's embrace that approach fully. The main advantages of running etcd in containers is that upgrading the cluster becomes as easy as executing `docker pull && docker run`. But we all agree, running ETCD without a scheduler can be a bit hairy depending on the OS the host runs. Again, if you ask for advise, we can say that most of the production installation we have seen are using CoreOS.

Running the Master in containers is also a great way of standarising the cluster. To do so, you just have to run `kubelet` and let kubelet to create the pods for the `api server`, the `scheduler` and the `control manager` (and any add-ons you want to use like DNS).

Finally, we use the same pattern to run our worker nodes. We start `kubelet` passing the `api server` url and off we go!
