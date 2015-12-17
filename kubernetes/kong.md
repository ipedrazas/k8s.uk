date: 2015-12-17
author: Álex González

Using Kong with Kubernetes
==========================

If you don't know about [Kong](https://getkong.org) yet you should take a look. It's an Open Source API Gateway, they define themselves as: "The open-source management layer for APIs, delivering high performance and reliability." and they are quite right.

I was playing with Kong lately at work ([jobandtalent.com](http://jobandtalent.com), we are hiring!) and I think that it could be pretty awesome as a entry layer to your microservices platform running in Kubernetes.

For the sake of simplicity I will not run Kong in Kubernetes, but it shouldn't be so difficult since [they already provide Docker images](https://getkong.org/install/). Also, running Kong on the same cluster you will be able to use internal networking between pods: win-win.

So, what will I show?

- I will deploy a Kubernetes with 2 pods (our 2 microservices) &
- I will install Kong locally and configure it to point to this 2 services.

Go & packing
------------

I've created a small Go app that will show the value of an environment variable when you `GET /`:

    package main

    import (
      "fmt"
      "log"
      "net/http"
      "os"
    )

    func main() {
      http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, os.Getenv("TEST_RESULT"))
      })

      log.Fatal(http.ListenAndServe(":8080", nil))
    }

Now we will build the application to later pack it in our image. Remember that if you are in Mac you will need to cross-compile the app to work on Linux:

    $ GOOS=linux go build

We can pack it into our images now. For doing so we need a `Dockerfile`. It's a simple binary, so the `Dockerfile` is not complex at all:

    FROM scratch
    ADD app /
    ENTRYPOINT ["/app"]

Cool! What can we do now with our shiny image? Yes, you are right! Push it to the hub:

    $ docker build -t agonzalezro/kong-test .
    $ docker push agonzalezro/kong-test

k8s
---

We have our image on the registry and all we need now is running it on Kubernetes. I am using [Google Container Engine](https://cloud.google.com/container-engine/) for deploying this, but you can use whatever you prefer.

Let's create our RCs & services:

    # rc1.yml
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: api1
    spec:
      selector:
        name: api
        version: first
      template:
        metadata:
          labels:
            name: api
            version: first
        spec:
          containers:
            - name: app
              image: agonzalezro/kong-test
              env:
                - name: TEST_RESULT
                  value: "This is the first app"

    # rc2.yml
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: api2
    spec:
      selector:
        name: api
        version: second
      template:
        metadata:
          labels:
            name: api
            version: second
        spec:
          containers:
            - name: app
              image: agonzalezro/kong-test
              env:
                - name: TEST_RESULT
                  value: "Second!"

    # svc1.yml
    apiVersion: v1
    kind: Service
    metadata:
      name: app1-svc
    spec:
      type: LoadBalancer
      ports:
        - port: 80
          targetPort: 8080
      selector:
        name: api
        version: first

    # svc2.yml
    apiVersion: v1
    kind: Service
    metadata:
      name: app2-svc
    spec:
      type: LoadBalancer
      ports:
        - port: 80
          targetPort: 8080
      selector:
        name: api
        version: second

And now run it:

    $ kubectl create -f rc1.yml -f rc2.yml -f svc1.yml -f svc2.yml

Wait for the service and the pods to be ready and check their IPS:

    $ kubectl get services
    NAME         CLUSTER_IP      EXTERNAL_IP      PORT(S)   SELECTOR                  AGE
    app1-svc     10.159.242.86   130.211.89.175   80/TCP    name=api,version=first    17m
    app2-svc     10.159.246.93   104.155.53.175   80/TCP    name=api,version=second   17m
    kubernetes   10.159.240.1    <none>           443/TCP   <none>                    1h

Kong
----

Follow the instruction here: https://getkong.org/install/docker/ to install Kong locally.

Yeah! We have it up & running so let's point it to our shinny cluster. We need to use Kong API for that (port `:8001`):

    $ http http://dockerhost:8001/apis/ name=first upstream_url=http://130.211.89.175 request_path=/first strip_request_path=true
    $ http http://dockerhost:8001/apis/ name=second upstream_url=http://104.155.53.175 request_path=/second strip_request_path=true

What we did here? We set up two new endpoints `/first` & `/second` that are pointing to the both Kubernetes services previously created. We could have done it with DNS as well using `request_host` instead.

Lets call Kong on the port `:8000` to use them:

    $ http http://dockerhost:8000/first
    HTTP/1.1 200 OK
    Connection: keep-alive
    Content-Length: 21
    Content-Type: text/plain; charset=utf-8
    Date: Thu, 17 Dec 2015 21:43:41 GMT
    Via: kong/0.5.4

    This is the first app

    $ http http://dockerhost:8000/second
    HTTP/1.1 200 OK
    Connection: keep-alive
    Content-Length: 7
    Content-Type: text/plain; charset=utf-8
    Date: Thu, 17 Dec 2015 21:43:44 GMT
    Via: kong/0.5.4

    Second!

\o/ We did it!

Next steps
----------

You have Kong pointed to your cluster, now it's up to your imagination what to do next. I would say try to configure some rate limiting or auth, it's deadly simply. Check them here: https://getkong.org/plugins/
