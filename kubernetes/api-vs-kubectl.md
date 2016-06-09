date: 2016-06-09
author: Ivan Pedrazas
tags: security, api, utils

Kubectl vs HTTP API
==========================

One of the best things Kubernetes has is its API, however, I've seen a few tools that instead of using the HTTP API use a wrapper on `kubectl`. I tweeted about it and a discussion was created around the differences between `kubectl` and the `HTTP API`.

This is the tweet:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">I makes me a bit sad to see tooling around `kubectl` when Kubernetes has such a nice HTTP API...</p>&mdash; Ivan Pedrazas (@ipedrazas) <a href="https://twitter.com/ipedrazas/status/740904502845411328">June 9, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

One thing that I hope it's clear it's that kubectl is designed to be used by people and HTTP API is designed to be used by code. In fact, if you look at the [documentation](http://kubernetes.io/docs/api/) you will see that there's a list of differnt APIs and kubectl is under `kubectl CLI`, this is teh list of all the kubernetes APIs:

* Kubernetes API
* Extension API
* Autoscaling API
* Batch API
* kubectl CLI

So, let's see what these differences are!

In order to create this test, I've spinned a new cluster in AWS. To be able to make remote calls to the API server you either need the token or the user and password for the basic auth:


    root@ip-172-20-0-9:~# cat /srv/kubernetes/basic_auth.csv
    PFK15d8JoTczqb3T,admin,admin

    # We need to base64 encode the user/password

    echo -n "admin:PFK15d8JoTczqb3T" | base64

    root@ip-172-20-0-9:~# cat /srv/kubernetes/known_tokens.csv
    iHNCmbTQCM4DRoN66PwEDqTFo2RA7JtJ,admin,admin
    59OcyDFhgBlqutaWzU7jxxMSAbVQs3oU,kubelet,kubelet
    jCbn8QSROs0gE525I7MIl0G1V7TEsXT4,kube_proxy,kube_proxy


__Token:__ `iHNCmbTQCM4DRoN66PwEDqTFo2RA7JtJ,admin,admin`

__Basic Auth:__ `YWRtaW46UEZLMTVkOEpvVGN6cWIzVA==`

__API_SERVER__: `https://52.51.69.149`

Let's export these values so we can re-use them easily:

    export TOKEN=iHNCmbTQCM4DRoN66PwEDqTFo2RA7JtJ
    export AUTH=YWRtaW46UEZLMTVkOEpvVGN6cWIzVA==
    export API_SERVER=https://52.51.69.149

Let's test that the authentication works:

    -> % curl -k -X GET -H "Authorization: Bearer $TOKEN" $API_SERVER
        {
          "paths": [
            "/api",
            "/api/v1",
            "/apis",
            "/apis/apps",
            "/apis/apps/v1alpha1",
            "/apis/autoscaling",
            "/apis/autoscaling/v1",
            "/apis/batch",
            "/apis/batch/v1",
            "/apis/batch/v2alpha1",
            "/apis/extensions",
            "/apis/extensions/v1beta1",
            "/apis/policy",
            "/apis/policy/v1alpha1",
            "/apis/rbac.authorization.k8s.io",
            "/apis/rbac.authorization.k8s.io/v1alpha1",
            "/healthz",
            "/healthz/ping",
            "/logs/",
            "/metrics",
            "/swaggerapi/",
            "/ui/",
            "/version"
          ]
        }

We will do basic auth also, for completion:

    curl -k -X GET -H "Authorization: Basic $AUTH" $API_SERVER

### List of Nodes

Kubectl:

    kubectl get nodes

API:

    curl -k -X GET -H "Authorization: Bearer $TOKEN" $API_SERVER/api/v1/nodes

If you have `jq` this command will be more useful:

    curl -k -X GET -H "Authorization: Bearer $TOKEN" $API_SERVER/api/v1/nodes | jq '.items[].status.addresses


Let's get a list of pods:

    kubectl get pods

This command returns nothing because we haven't created a single object yet... but, we can do:

    kubectl get pods --all-namespaces


    NAMESPACE     NAME                                                               READY     STATUS             RESTARTS   AGE
    kube-system   elasticsearch-logging-v1-7n3fn                                     1/1       Running            0          1h
    kube-system   elasticsearch-logging-v1-qcr9m                                     1/1       Running            0          1h
    kube-system   fluentd-elasticsearch-ip-172-20-0-166.eu-west-1.compute.internal   1/1       Running            0          1h
    kube-system   fluentd-elasticsearch-ip-172-20-0-167.eu-west-1.compute.internal   1/1       Running            0          1h
    kube-system   heapster-v1.1.0.beta2-2783873945-91yz7                             4/4       Running            0          1h
    kube-system   kibana-logging-v1-f4q1e                                            0/1       CrashLoopBackOff   16         1h
    kube-system   kube-proxy-ip-172-20-0-166.eu-west-1.compute.internal              1/1       Running            0          1h
    kube-system   kube-proxy-ip-172-20-0-167.eu-west-1.compute.internal              1/1       Running            0          1h
    kube-system   kubernetes-dashboard-v1.1.0-beta1-imh41                            1/1       Running            0          1h
    kube-system   monitoring-influxdb-grafana-v3-12tud                               2/2       Running            0          1h

Excellent, let's do the http query now:

    curl -k -X GET -H "Authorization: Bearer $TOKEN" $API_SERVER/api/v1/pods

Result is pretty large, so, [let's gist it](https://gist.github.com/ipedrazas/0baf49dd82db6730db8fab9816353d3c):

    {
      "kind": "PodList",
      "apiVersion": "v1",
      "metadata": {
        "selfLink": "/api/v1/pods",
        "resourceVersion": "1172"
      },
      "items": [
        {
         "metadata": {
         "name": "elasticsearch-logging-v1-7n3fn",
         "generateName": "elasticsearch-logging-v1-",
         "namespace": "kube-system",
         "selfLink": "/api/v1/namespaces/kube-system/pods/elasticsearch-logging-v1-7n3fn",
         "uid": "19617fb6-2e4d-11e6-a8cc-0a800fdf3429",
         ...

Ok, let's create a namespace, then. The namespace will be defined in a file `new_namespace.yaml` with the content:

    apiVersion: v1
    kind: Namespace
    metadata:
      name: api-vs-kubectl

All the files used in this post can be found in [our github repo](https://github.com/ipedrazas/k8s.uk/tree/master/code/api-vs-kubelet)
Now, let's create the namespace:

    kubectl create -f new_namespace.yaml

Now via HTTP API

    curl -k -H "Content-Type: application/yaml" -H "Authorization: Bearer $TOKEN" -XPOST -d"$(cat api_ns.yaml)" $API_SERVER/api/v1/namespaces
    {
      "kind": "Namespace",
      "apiVersion": "v1",
      "metadata": {
        "name": "api",
        "selfLink": "/api/v1/namespaces/api",
        "uid": "aee75ba2-2e5a-11e6-a8cc-0a800fdf3429",
        "resourceVersion": "1421",
        "creationTimestamp": "2016-06-09T15:56:09Z"
      },
      "spec": {
        "finalizers": [
          "kubernetes"
        ]
      },
      "status": {
        "phase": "Active"
      }

Now, let's create some containers:

    kubectl create -f kubectl_nginx.yaml --namespace=kubectl

With our API:

    curl -k -H "Content-Type: application/yaml" -H "Authorization: Bearer $TOKEN" -XPOST -d"$(cat api_nginx.yaml)" $API_SERVER/api/v1/namespaces/api/replicationcontrollers


Now, if we pull all the pods we will see that we have the same exepected results:

    NAMESPACE     NAME                                                               READY     STATUS             RESTARTS   AGE
    kube-system   elasticsearch-logging-v1-7n3fn                                     1/1       Running            0          1h
    kube-system   elasticsearch-logging-v1-qcr9m                                     1/1       Running            0          1h
    kube-system   fluentd-elasticsearch-ip-172-20-0-166.eu-west-1.compute.internal   1/1       Running            0          1h
    kube-system   fluentd-elasticsearch-ip-172-20-0-167.eu-west-1.compute.internal   1/1       Running            0          1h
    kube-system   heapster-v1.1.0.beta2-2783873945-91yz7                             4/4       Running            0          1h
    kube-system   kibana-logging-v1-f4q1e                                            0/1       CrashLoopBackOff   25         1h
    kube-system   kube-proxy-ip-172-20-0-166.eu-west-1.compute.internal              1/1       Running            0          1h
    kube-system   kube-proxy-ip-172-20-0-167.eu-west-1.compute.internal              1/1       Running            0          1h
    kube-system   kubernetes-dashboard-v1.1.0-beta1-imh41                            1/1       Running            0          1h
    kube-system   monitoring-influxdb-grafana-v3-12tud                               2/2       Running            0          1h
    kubectl       nginx-we1fb                                                        1/1       Running            0          6m
    api           nginx-7nnad                                                        1/1       Running            0          46s

It's clear that the CRUD is covered by both, the API and kubectl, let's scale up/down pods:

    kubectl scale rc nginx --replicas=3 --namespace=kubectl
    NAME          READY     STATUS    RESTARTS   AGE
    nginx-onnme   1/1       Running   0          8s
    nginx-p3zwa   1/1       Running   0          8s
    nginx-we1fb   1/1       Running   0          8m


With the API, we're using PUT, that updates the object, so, we have to re-submit the modified object

    curl -k -H "Content-Type: application/yaml" -H "Authorization: Bearer $TOKEN" -XPUT -d"$(cat api_nginx-3.yaml)" $API_SERVER/api/v1/namespaces/api/replicationcontrollers/nginx


What about watching resources for changes?

    watch kubectl get pods

With the API:

    curl -k -X GET -H "Authorization: Bearer $TOKEN"   $API_SERVER/api/v1/pods?watch=true

In summary, kubectl is a great tool to interact with kubernetes, but remember that there's also a fantastic API that allows you to interact with the cluster also.

What are the main differences? Well, a person should use `kubectl` but a system, or an application would use the API. In fact, if you're making a tool that interacts with kubernetes and you're using `kubectl` under the hood, I bet it's because you feel more confortable with kubectl and not so much with the REST API... But nothing comes free, you trade the knowledge you have of a tool by having to parse text instead of clearly defined objects that the API should return.

Truth is, that the API documentation is not great either, so... maybe we should ask to have a bunch of examples that use the API so people can learn faster :)
