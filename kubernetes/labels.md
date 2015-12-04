Date: 2015-12-04 15:38
tags: management, labels, core

Deployments powered by Labels
======================

Labels in kubernetes are the only way of grouping, filtering and selecting resources. In my experience, we do not use labels enough. Unlike with AWS, you can define as many labels as you want, so, why are we not truly squeezing all the potential of labels?

I'd say it's because we're not use to, and because we lack clear use cases. Yes, it drills down to we getting used to organise and manage things in a specific way.

In my experience, deployments is one of the parts where being smart with your labels can take you a very long way.

Have you ever done a deployment where everything went well but for a non technical issue you had to roll back? something like "Oh, God, that promotion cannot go live just yet... ROLLBACK! ROLLBACK!!"

Labels are key/value pairs. I usually tag or label my resources with:

* __state__: (production, test, lab...)
* __version__: (the version of the app following [SemVer](http://semver.org/))
* __channel__:  ("public" if it's accessible from the internet, "internal", "cannary")
* __owner__: who owns the resource. this can be for making your life easier when doing interal billing, or for knowing who to get in contact with if something goes wrong with it.

So, how do you deploy a new version of your application?

Let's assume the initial state is this:

* Replication Controller: deep-frontend, replicas: 3
* Pod: deep-reports-38x6n (state: production, version: 1.0.7, channel: public, owner: ipedrazas)
* Service: selector state=production, channel=public

Note that your app is available through the service and the service binds to the pods that have the labels `state=production`,  `channel=public` and `version= 1.0.7`.

If now you deploy a new Replication controller you will have the following scenario:

* Replication Controller: deep-frontend, replicas: 3
* Replication Controller: deep-frontend_new, replicas: 3
* Pod: deep-frontend-38x6n (state: production, version: 1.0.7, channel: public, owner: ipedrazas)
* Pod: deep-frontend_new-jag1p (state: production, version: 1.0.8, channel: public, owner: ipedrazas)
* Service: selector state=production, channel=public, version=1.0.7

You can see where I'm going. You can then go and update a label in the service: `version=1.0.8` and your service is repointed to the new app. As you can see, the old app is still in the system, rolling back is just a matter of updating the label back to its original value.

Production deployments have a very wide range of situations. Trying to find a golden rule for deployments (standarisation, standarisation) usually doesn't work well, so it's better to define certain patterns that you can adapt to your needs.

Labels are very powerful, and usually they're not used to their maximum potential. The post doesn't try to convince you to change your deployment strategy but to illustrate different ways of doing the same thing, and different use cases of this kubernetes artifact that can change the way you understand your landscape.
