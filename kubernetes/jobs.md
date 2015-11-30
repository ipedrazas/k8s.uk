Date: 2015-11-30 12:08
tags: job, kubernetes, core

Jobs
======================

No, this post is not about recruitment, it's about the [__Jobs__ artifact](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/user-guide/jobs.md) that was introduced with version 1.1.

Jobs are the way in Kubernetes of running on-off containers (or pods). Until now, to run a container you could define a pod and schedule it without a replication controller. Now, we have the concept of a "job". In reality, "Task" would be more appropriate, but as we can see in the roadmap, scheduling jobs will come in version 1.2.

If we read the definition in the docs it says:

    A job creates one or more pods and ensures that a specified number of them successfully terminate. As pods successfully complete, the job tracks the successful completions. When a specified number of successful completions is reached, the job itself is complete. Deleting a Job will cleanup the pods it created.

So, What can we do with jobs now? One of the things we do is to run Integration Test. You deploy your artifacts: pods, replication controllers, and servcies and then, we run a job to test that everything works as expected.

Building a CI pipeline is hard. Using kubernetes simplifies a bit certain aspects (like the deployment) but it makes harder other bits (like knowing when everything is ready). We're preparing a big post about CI, so, stay tunned.

A job definition is very similar to a Replication controller or a Pod:


      apiVersion: extensions/v1beta1
      kind: Job
      metadata:
        name: pi
      spec:
        selector:
          matchLabels:
            app: pi
        template:
          metadata:
            name: pi
            labels:
              app: pi
          spec:
            containers:
            - name: pi
              image: perl
              command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
            restartPolicy: Never


Note that the __apiVersion__ uses [extensions](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/api.md#api-groups). The schema of a job will change once the scheduling features land in version 1.2, but for now, it's very straight forward: define the artifact kind as a job, include the pod template definition and off you go. Another thing to note is the __restartPolicy__, you can set it to Always, OnFailure, or Never, depending on what you think it's appropriate.



