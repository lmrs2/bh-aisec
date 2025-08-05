# 04 - Admission controller

## Highlight

Let's start by familiarizing ourselves with the goal of this activity.

### Attacks prevented

Enforcing a deployment policy before deploying packages mitigate an attack not currently mentioned in the diagram which we could call (I) "deploy unauthorized package", e.g., a debug package to a prod environment, or a malicious (integrity protected) package to a prod environment.

### What you will need

Install the [necessary software](https://github.com/lmrs2/bh-aisec/blob/main/INSTALLATION.md).

### Admission controller

The admission controller is the component that deploys artifacts / containers. It needs to be configured to verify deployment attestations. Verification requires the following metadata:

1. Trusted roots, which is the metadata that defines:
    1. Which evaluators we trust - defined by their identity.
    1. Which protection type (e.g., service account, cluster ID) each evaluator is authoritative for.
    1. Required protection types, which is an optional set of mandatory protection types. To be considered authentic and trusted, a deployment attestation must contain the required protection types. 
1. Mode of enforcement, such as "enforce" or "audit". This allows administrators to onboard new teams and roll out policy upgrades in stages.
1. Failure handling, which configures how unexpected errors or timeouts during evaluation are handled. Fail open behavior admits deployments by default in such a scenario, whereas fail closed behavior defaults to a rejection. The low latency and reliability of using deployment attestations should make these occurrences rare in comparison to real time evaluation

For example, a trusted root could contain (a) public key pubKeyA as evaluator identity, (b) be authoritative for protection types "google service account" and "Kubernetes namespace" and (c) have the scope type "google service account" required.  A deployment attestation is considered authentic and trusted if (a) it is signed using pubKeyA, (c and b) contains the protection types "google service account" and (b) may optionally contain a scope of type "Kubernetes namespace".

In this workshop, we use the open source policy engine [Kyverno](https://kyverno.io/).

## Deep dive

### Installation

#### cosign

You should already have cosign installed as required for Activities [02](https://github.com/lmrs2/bh-aisec/tree/main/activities/02) and [03](https://github.com/lmrs2/bh-aisec/tree/main/activities/03). If that's not the case, follow the [installation instructions](https://github.com/lmrs2/bh-aisec/blob/main/INSTALLATION.md#cosign).

#### Local Kubernetes

In this demo, we use a local Kubernetes installation called [minikube](https://minikube.sigs.k8s.io/docs/start/).

To install minikube, follow the [instructions](https://github.com/lmrs2/bh-aisec/blob/main/INSTALLATION.md#minikube).

Start minikube:

```shell
# Kyverno 1.11.4 supports Kubernetes version v1.25-v.18, see https://kyverno.io/docs/installation/#compatibility-matrix.
$ minikube start --kubernetes-version=v1.28
```

Set the alias:

```shell
$ alias kubectl="minikube kubectl --"
```

#### Kyverno policy engine

Install [Kyverno policy engine](https://github.com/lmrs2/bh-aisec/blob/main/INSTALLATION.md#kyverno):

IMPORTANT: Open a new terminal and monitor the logs for the admission controller and keep this terminal open:

```shell
# Replace 'kyverno-admission-controller-6dd8fd446c-4qck5' with the value for your installation (previous command).
$ kubectl -n kyverno logs -f kyverno-admission-controller-6dd8fd446c-4qck5
```

### Admission controller configuration

We need to configure Kyverno to verify the deployment attestation we created in [Activity 03](https://github.com/lmrs2/bh-aisec/blob/main/activities/03/readme.md).

There are two relevant files to configure it:

1. A verification configuration file containing the trusted roots, in [kyverno/slsa-configuration.yml](https://github.com/lmrs2/bh-aisec-project1/blob/main/kyverno/slsa-configuration.yml).
1. A Kyverno enforcer file [kyverno/slsa-enforcer.yml](https://github.com/lmrs2/bh-aisec-project1/blob/main/kyverno/slsa-enforcer.yml) that verifies the deployment attestation using the trusted roots.

Clone the repository locally. Then follow the steps:

1. Update the [attestation_creator](https://github.com/lmrs2/bh-aisec-project1/blob/main/kyverno/slsa-configuration.yml#L16) field in the verification configuration file. Set it to the value of the evaluator identity that created your deployment attestation in [Activity 03](https://github.com/lmrs2/bh-aisec/blob/main/activities/03/readme.md).
1. Install the policy engine

```shell
$ kubectl apply -f kyverno/slsa-configuration.yml
$ kubectl apply -f kyverno/slsa-enforcer.yml
```

### Deploy a pod

Let's deploy the container we built in [Activity 01](https://github.com/lmrs2/bh-aisec/blob/main/activities/01/readme.md). For that, we will use the [k8/echo-server-deployment.yml](https://github.com/lmrs2/bh-aisec-project1/blob/main/k8/echo-server-deployment.yml) pod definition.


Follow these steps:

1. Edit the [image](https://github.com/lmrs2/bh-aisec-project1/blob/main/k8/echo-server-deployment.yml#L23) in the pod definition.
1. WARNING: Since we are running Kubernetes locally, there is no google service account to match against. To simulate one exists for our demo, we make the assumption that its value is exposed via the ["cloud.google.com.v1/service_account" annotation](https://github.com/lmrs2/bh-aisec-project1/blob/main/k8/echo-server-deployment.yml#L18). Set the value to the service account configured in your deployment policy for the container.
1. Deploy the container

```shell
$ kubectl apply -f k8/echo-server-deployment.yml
deployment.apps/echo-server-deployment created
```

Make sure all the pods are in a running state:

```shell
kubectl get po -A
```

Run the following commands to confirm the deployment succeeded:

```shell
$ kubectl get polr
NAME                                   KIND         NAME                                      PASS   FAIL   WARN   ERROR   SKIP   AGE
13f78700-4f91-44e6-aa1b-970ed83251dc   ReplicaSet   echo-server-deployment-5bcdd7d764         1      0      0      0       0      4m5s
6b8c9fe2-89fb-4388-92e3-67abdaf3feb0   Pod          echo-server-deployment-5bcdd7d764-87cxt   1      0      0      0       0      4m35s
8863a504-63d6-4455-b9dd-79e15f2bd75f   Pod          echo-server-deployment-5bcdd7d764-2rrrm   1      0      0      0       0      4m35s
977d4976-7ce3-4d97-861a-8a119f3c5e84   Pod          echo-server-deployment-5bcdd7d764-27h96   1      0      0      0       0      4m35s
c482b133-13b1-4678-bb2c-0de2d44c868d   Deployment   echo-server-deployment                    1      0      0      0       0      4m36s
```

Get the service URL:

```shell
$ minikube service echo-server-service --url
http://127.0.0.1:63111
```

Send a command to the service:

```shell
$ curl -s -X POST -H "Content-Type: application/json"      -d '{"image": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII="}' http://127.0.0.1:63111/classify/v0
```

You should see a JSON file with the decoded image in hexadecimal. (In the next activity we will add support for the model inference.)

Now update the pod definition with an image that is _not_ allowed to run under this service account:

1. Edit the [image](https://github.com/lmrs2/bh-aisec-project1/blob/main/k8/echo-server-deployment.yml#L23) in the deployment file. 

```shell
$ kubectl apply -f k8/echo-server-deployment.yml
...THIS SHOULD FAIL... WATCH OUT YOUR LOGS
```

Update the pod definition back to its original value.

1. Edit the ["cloud.google.com.v1/service_account" annotation](https://github.com/lmrs2/bh-aisec-project1/blob/main/k8/echo-server-deployment.yml#L18) to a different service account.

```shell
$ kubectl apply -f k8/echo-server-deployment.yml
...THIS SHOULD FAIL... WATCH OUT YOUR LOGS
```

### Future work

#### Limitation

To our knowledge, the google service account is not available to Google's GKE. One way to deploy a real-world example
of this demo is to [bind Kubernetes service account to a google service account](https://github.com/GoogleCloudPlatform/community/blob/master/archived/restrict-workload-identity-with-kyverno/index.md). This is out of scope of the workshop and we leave if for future work. If you take on this task, please share the code with us!

#### Support other protection types

Can you update the policy engine to support other types of protections, e.g., GKE cluster ID, etc. 

## Take the quizz!

After completing this activity, you should be able to answer the following questions:

1. What metadata is needed to configure the admission controller?
