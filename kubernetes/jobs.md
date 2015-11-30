Date: 2015-11-30 12:08

Jobs
======================

No, this post is not about recruitment, it's about the __Jobs__ artifact that was introduced with version 1.1.

Jobs is the way in Kubernetes of running on-off containers (or pods). Until now, to run a container you could define a pod and schedule it without a replication controller. Now, we have the concept of a "job". In reality, "Task" would be more appropriate, but as we can see in the roadmap, scheduling jobs will come in version 1.2.

So, What can we do with jobs now? One of the things we do is to run Integration Test. You deploy your artifacts: pods, replication controllers, and servcies and then, we run a job to test that everything works as expected.

Building a CI pipeline is hard. Using kubernetes simplifies a bit certain aspects (like the deployment) but it makes harder other bits (like knowing when everything is ready). We're preparing a big post about CI, so, stay tunned.

A job definition is very similar to a Replication controller or a Pod:


        {
          "apiVersion": "extensions/v1beta1",
          "kind": "Job",
          "metadata": {
            "name": "healthchecker",
            "labels": {
              "name": "healthchecker",
              "deep": "healthchecker"
            }
          },
          "spec": {
            "template": {
              "metadata": {
                "name": "healthchecker",
                "labels": {
                  "name": "healthchecker",
                  "deep": "healthchecker"
                }
              },
              "spec": {
                "restartPolicy": "Never",
                "containers": [
                  {
                    "name": "healthchecker",
                    "image": "ipedrazas/healthchecker:latest",
                    "env": [
                        {
                          "name": "HOOK",
                          "value": "http://deep-api.ipedrazas.k8s.co.uk:5000/checks"
                        },
                        {
                          "name": "APIHOST",
                          "value": "http://deep-api.ipedrazas.k8s.co.uk:5000/healthchecks"
                        }
                   ]
                 }
                ]
              }
            }
          }
        }

Note that the __apiVersion__ uses extensions. The schema of a job will change once the scheduling features land in version 1.2, but for now, it's very straight forward: define the artifact kind as a job, include the pod template definition and off you go. Another thing to note is the __restartPolicy__


