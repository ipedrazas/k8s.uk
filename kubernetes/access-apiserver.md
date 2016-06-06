date: 2016-06-06
author: Ivan Pedrazas
tags: security, tokens

Accessing Kubernetes Apiserver
==========================

The process to access the api server is very simple. The `apiserver` has a flag that defines what type of access is desired:

* --authorization-mode=AlwaysDeny blocks all requests (used in tests).
* --authorization-mode=AlwaysAllow allows all requests; use if you don’t need authorization.
* --authorization-mode=ABAC allows for user-configured authorization policy. ABAC stands for Attribute-Based Access Control.
* --authorization-mode=Webhook allows for authorization to be driven by a remote service using REST.

To allow Basic Auth and/or tokens, we have to select `ABAC`.


## Access

To access the API server via tokens there are 2 things that need to be defined: the token/user and what the user is allowed to do. Tokens are defined in a file, policies are defined in a different file.

These configuration files have to be passed to the `kube-apiserver` using the following parameters:

* --authorization-mode=ABAC
* --token-auth-file=/srv/kubernetes/auth_tokens.csv
* --authorization-policy-file=/srv/kubernetes/auth-policy.json

If you want to allow Basic Auth, you have to specify the file containing the

* --basic-auth-file=/srv/kubernetes/basic_auth.csv


Example of running the apiserver with those flags:

        /bin/sh -c /usr/local/bin/kube-apiserver --address=127.0.0.1 --etcd-servers=http://127.0.0.1:4001
        --admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,PersistentVolumeLabel,ResourceQuota
        --token-auth-file=/srv/kubernetes/auth_tokens.csv
        --authorization-mode=ABAC
        --authorization-policy-file=/srv/kubernetes/auth-policy.json
        --basic-auth-file=/srv/kubernetes/basic_auth.csv

Here are examples of the files used by the `apiserver`:

Example of tokens in `auth_tokens.csv`:

        Wx4WOTOmFoY5yXaoMPtHdnKeFLPYeBBL,admin,admin
        jD34eFwNrJo9urd7QWLMALOjK7R58j1g,kubelet,kubelet
        PxgQ4vSloVIhfFwx9WYaj8uke93JVBHh,kube_proxy,kube_proxy
        i2TgpiZFZQNkIydDZzVkxmTHl3Q2hPNn,ivan,ivan

Example of user/password for Basic Auth `basic_auth.csv`:

        GGIfwZn63i3NMWeN,admin,admin

Example of authentication policy file  `auth-policy.json`

        {"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"ivan", "namespace": "*", "resource": "*", "apiGroup": "*", "nonResourcePath": "*"}}
        {"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"admin", "namespace": "*", "resource": "*", "apiGroup": "*", "nonResourcePath": "*"}}
        {"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"kubelet", "namespace": "*", "resource": "*", "apiGroup": "*", "nonResourcePath": "*"}}
        {"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"kube_proxy", "namespace": "*", "resource": "*", "apiGroup": "*", "nonResourcePath": "*"}}
        {"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"kubecfg", "namespace": "*", "resource": "*", "apiGroup": "*", "nonResourcePath": "*"}}
        {"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"client", "namespace": "*", "resource": "*", "apiGroup": "*", "nonResourcePath": "*"}}
        {"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:serviceaccounts", "namespace": "*", "resource": "*", "apiGroup": "*", "nonResourcePath": "*"}}


## Testing

If you want to test the access, you can try the following commands:

Access using Basic Auth:

    curl -k -X GET -H   "Authorization: Basic YWRtaW46R0dJZndabjYzaTNOTVdlTg=="   https://$API_SERVER

Note that the string `YWRtaW46R0dJZndabjYzaTNOTVdlTg==` is the result of

    echo -n "admin:GGIfwZn63i3NMWeN" | base64

Access using tokens:

    curl -k -X GET -H "Authorization: Bearer i2TgpiZFZQNkIydDZzVkxmTHl3Q2hPNn"    https://$API_SERVER


### Policy File Format

For mode  __ABAC__, also specify `--authorization-policy-file=SOME_FILENAME`.
The file format is one JSON object per line. There should be no enclosing list or map, just one map per line.
Each line is a “policy object”. A policy object is a map with the following properties:

* Versioning properties:
    * __apiVersion__, type string; valid values are “abac.authorization.kubernetes.io/v1beta1”. Allows versioning and conversion of the policy format.
    * __kind__, type string: valid values are “Policy”. Allows versioning and conversion of the policy format.
* spec property set to a map with the following properties:
    * Subject-matching properties:
        * __user__, type string; the user-string from `--token-auth-file`. If you specify user, it must match the username of the authenticated user. * matches all requests.
        * __group__, type string; if you specify group, it must match one of the groups of the authenticated user. * matches all requests.
    * __readonly__, type boolean, when true, means that the policy only applies to get, list, and watch operations.
    * Resource-matching properties:
        * __apiGroup__, type string; an API group, such as extensions. * matches all API groups.
        * __namespace__, type string; a namespace string. * matches all resource requests.
        * __resource__, type string; a resource, such as pods. * matches all resource requests.
    * Non-resource-matching properties:
        * nonResourcePath, type string; matches the non-resource request paths (like /version and /apis). * matches all                * __non-resource requests__. /foo/* matches /foo/ and all of its subpaths.

