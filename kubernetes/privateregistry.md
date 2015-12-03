Date: 2015-12-3 09:58
tags: registry, core

Private Registry
======================

The first time we tried to use a private registry in Kubernetes we got bitten by a weird bug: the format of the `.dockercfg`.

If you read the [documentation](http://kubernetes.io/v1.1/docs/user-guide/images.html) you will see that you have to create a secret, and then use that secret in your pod definition.

What it seems to be missing from the docs is that the format of the json that contains the registry auth info is important.

This is the example in the documentation:

    $ echo $(cat ~/.dockercfg)
    { "https://index.docker.io/v1/": { "auth": "ZmFrZXBhc3N3b3JkMTIK", "email": "jdoe@example.com" } }

But we need base64 encode text:

    $ cat ~/.dockercfg | base64
    eyAiaHR0cHM6Ly9pbmRleC5kb2NrZXIuaW8vdjEvIjogeyAiYXV0aCI6ICJabUZyWlhCaGMzTjNiM0prTVRJSyIsICJlbWFpbCI6ICJqZG9lQGV4YW1wbGUuY29tIiB9IH0K

Finally, we create the secret definiton using the base64

    $ cat > /tmp/image-pull-secret.yaml <<EOF
    apiVersion: v1
    kind: Secret
    metadata:
      name: myregistrykey
    data:
      .dockercfg: eyAiaHR0cHM6Ly9pbmRleC5kb2NrZXIuaW8vdjEvIjogeyAiYXV0aCI6ICJabUZyWlhCaGMzTjNiM0prTVRJSyIsICJlbWFpbCI6ICJqZG9lQGV4YW1wbGUuY29tIiB9IH0K
    type: kubernetes.io/dockercfg
    EOF

Finally, we create the secret in the cluster

    $ kubectl create -f /tmp/image-pull-secret.yaml
    secrets/myregistrykey

Once we have the secret, we can use that secret in our pods.

    apiVersion: v1
    kind: Pod
    metadata:
      name: foo
    spec:
      containers:
        - name: foo
          image: janedoe/awesomeapp:v1
      imagePullSecrets:
        - name: myregistrykey

This is all true and good, but the detail of the format of .dockercfg is the kind of thing that will have you running around calling names. So, what's the problem? If you execute the first command of the post:

    $ echo $(cat ~/.dockercfg)

You will see that it returns 1 line. However, between catting the file and echoing the catting fo the file there's one little detail: that ECHO makes the file to be in one single line.

Now, look at this:

    -> % cat .dockercfg| base64
    ewoKCSJodHRwczovL2luZGV4LmRvY2tlci5pby92MS8iOiB7CgkJImF2dGgiOiAiYVhCbFpISmhl
    bUZ6T25Rd2REQnlNRMV5T21SdiIsCgkJImVtYWlsIjogImlwZWRyYXphc0BnbWFpbC5jb20iCgl9
    Cn0=

    -> % echo $(cat .dockercfg) | base64
    eyAiaHR0cHM6Ly9pbmRleC5kb2NrZXIuaW8vdjEvIjogeyAiYXV2aCI6ICJhWEJsWkhKaGVtRnpP
    blF3ZERCeU1ERXlPbVJ2KiwgImVtYWlsIjogImlwZWRyYXphc0BnbWFpbC5jb20iIH0gfQo=


I'm not sure why nobody has bothered writing this little note in the docs, but if you don't echo the cat, the secret will be wrong. Anyway, from now on, remember, the format of the json is important... so make sure you verify the json you're using is in one single line.


