
# 05 - Support for model training and inference

## Highlight

In this final activity, we will

1. Train a model and generate SLSA provenance for it; publish it and generate a deployment attestation.
1. Update the echo-server code to predict digits from an image.
1. Deploy the server and the model to our k8 cluster.

### What you will need

Install the [necessary software](https://github.com/lmrs2/bh-aisec/blob/main/INSTALLATION.md).

## Building the inference container

We udate the python echo-server to perform digit predictions. For this, you must:

1. Uncomment [this line](https://github.com/lmrs2/bh-aisec-project1/blob/main/.github/workflows/build-echo-server.yml#L55) to set the Dockerfile to [Dockerfile_predict](https://github.com/lmrs2/bh-aisec-project1/blob/main/images/echo-server/Dockerfile_predict). Instead of the prior [app.py](https://github.com/lmrs2/bh-aisec-project1/blob/main/images/echo-server/app.py), it builds the [app_predict.py](https://github.com/lmrs2/bh-aisec-project1/blob/main/images/echo-server/app_predict.py).
1. Build the container as in [Activity 01](https://github.com/laurentsimon/bh-aisec/tree/main/activities/01).
1. Publish the container as in [Activity 02](https://github.com/laurentsimon/bh-aisec/tree/main/activities/02)
1. Generate the publish attestation as in [Activity 03](https://github.com/laurentsimon/bh-aisec/tree/main/activities/01).

## Training the model

### Create the workflow

Fork this repository [https://github.com/lmrs2/bh-aisec-model](https://github.com/lmrs2/bh-aisec-model) by clicking this [link](https://github.com/lmrs2/bh-aisec-model/fork). Navigate to the Actions tab and enable worksflows, as depicted in the image below:

![workflow image enable](https://raw.githubusercontent.com/lmrs2/bh-aisec/main/images/enable-workflows.jpg "How to enable workflows in your forked repository")

The repository contains the code to [train a digit classifier using MNIST dataset](https://github.com/lmrs2/bh-aisec-model/tree/main/training). The repository also contains a GitHub workflow [.github/workflows/train-model.yml](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/train-model.yml) which builds and generates provenance for the trained model. The workflow file contains the following steps:

1. [Train the model](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/train-model.yml#L30-L43), which generates a file [mnist_classifier.pth](https://github.com/lmrs2/bh-aisec-model/blob/main/training/train.py#L156).
1. [Store the model into a container](https://github.com/lmrs2/bh-aisec-model/blob/main/training/Dockerfile#L8). We store the file into a container to simplify the example. In practice, you may store the model on HuggingFace or in a cloud storage, or anywhere that suits your needs.
1. [Build the container](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/train-model.yml#L45-L71) (we use the steps to build the echo-server container in [Activity 01](https://github.com/lmrs2/bh-aisec/tree/main/activities/01)).
1. In a seperate job, we [call the container generator](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/train-model.yml#L79-L94) with the image name and digest. Note that it is IMPORTANT for the digest to be computed _without_ pulling the image from the registry, because an attacker / insider _could_ push a malicious image between our workflow push and pull.
1. As a sanity step, we [pull the container](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/train-model.yml#L117).

### Run the workflow

Follow these steps (similar to [Activity 01](https://github.com/lmrs2/bh-aisec/tree/main/activities/01)):

1. Update the [REGISTRY_USERNAME](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/train-model.yml#L15) to your own docker registry username.
1. Update the [hardcoded username ](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/train-model.yml#L90) used to store the provenance to registry.
1. Create a [docker registry token](https://docs.docker.com/security/for-developers/access-tokens/#create-an-access-token) with read, write and delete access.
2. Store your docker token as a new GitHub repository secret called `REGISTRY_PASSWORD`: [Settings > New repository secret](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository).
2. Run the workflow via the [GitHub UI](https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow#running-a-workflow). It will take ~2mn to complete. If all goes well, the workflow run will display a green icon. Click on the job run called "run" sttept "Run it" (see [example run](https://github.com/lmrs2/bh-aisec-model/actions/runs/16094432352)). Note the name of the container displayed in the logs. In the example above, it is `docker.io/lmrs2/bh-aisec-model-inference@sha256:b694e56c2c4c60a3e771d7dde95e26c282e1bd6a30a7e8a46c6dc986c54be906`.

### Verify provenance

Install the [slsa-verifier](https://github.com/slsa-framework/oss-na24-slsa-workshop/blob/main/INSTALLATION.md#slsa-verifier).

Make sure you have access to your image by authenticating to docker:

```shell
$ REGISTRY_TOKEN=<your-token>
$ REGISTRY_USERNAME=<registry-username>
$ docker login -u "${REGISTRY_USERNAME}" "${REGISTRY_TOKEN}"
```

To verify your container, use the following command:

```shell
# Update the image as recorded in your logs. You will sit it printed under the "run" build.
$ image=docker.io/lmrs2/bh-aisec-model-inference@sha256:b694e56c2c4c60a3e771d7dde95e26c282e1bd6a30a7e8a46c6dc986c54be906
# Replace with your own repository.
$ source_uri=github.com/lmrs2/bh-aisec-model
$ path/to/slsa-verifier verify-image "${image}" --source-uri "${source_uri}" --builder-id=https://github.com/slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml
```

## Publish the model

### Configure the policy

Like we did in [Activity 02](https://github.com/lmrs2/bh-aisec/tree/main/activities/02) for the echo-server, we need to create a publish attestation to publish our model.

The file [model-inference.json](https://github.com/lmrs2/bh-aisec-organization/blob/main/policies/publish/model-inference.json) (which, in a real delpoyment, would be protected by a CODEOWNER file), describes the team policy for the model we trained. The file contains the following sections:

1. The [package](https://github.com/lmrs2/bh-aisec-organization/blob/main/policies/publish/model-inference.json#L3) section describes the package to publish, i.e., [docker.io/lmrs2/bh-aisec-model-inference](https://github.com/lmrs2/bh-aisec-organization/blob/main/policies/publish/model-inference.json#L4) and will be used both for [staging and prod](https://github.com/lmrs2/bh-aisec-organization/blob/main/policies/publish/model-inference.json#L7). NOTE: The environment (prod, staging) is optional.
1. The [build](https://github.com/lmrs2/bh-aisec-organization/blob/main/policies/publish/model-inference.json#L11) section describes how to train the model, i.e. it must be trained using code from the source repository [github.com/lmrs2/bh-aisec-model](https://github.com/lmrs2/bh-aisec-organization/blob/main/policies/publish/model-inference.json#L14) by builder [github_generator_level_3](https://github.com/lmrs2/bh-aisec-organization/blob/main/policies/publish/model-inference.json#L12).


Follow these steps:

1. Update the value of the package name with your container image containing the trained model.
1. Update the value of the source respository.

### Call the evaluator in CI

To evaluate the publish policy, the evaluator must be called from CI. It is up to teams to decide _when_ to do that. In this activity, we provide a helper workflow
[.github/workflows/publish-model.yml](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/publish-model.yml) that can be called manually for testing purposes.
In practice, teams would call it automatically after building their containers.

If the policy evaluation succeeds, the evaluator creates a publish attestation and signs it with [Sigstore](sigstore.dev). The attestation is stored along the container on the registry, [a-la-cosign](https://github.com/sigstore/cosign). NOTE: SLSA does not prescribe where to store the provenance, nor does it prescribe the use of Sigstore for signing.

Follow these steps:

1. Update the [organization workflow call](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/publish-model.yml#L37) that evaluates the publish policy.
1. Update the [registry-username](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/publish-model.yml#L43) to yours.
1. (Already done in Activity 01): Create a [docker regitry token](https://docs.docker.com/security/for-developers/access-tokens/#create-an-access-token) with read, write and delete access. 
1. (Already done in Activity 01): Store your docker token as a new GitHub repository secret called `REGISTRY_PASSWORD`: [Settings > New repository secret](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository).
1. Run the workflow via the [GitHub UI](https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow#running-a-workflow). It will take ~40s to complete. If all goes well, the workflow run will display a green icon.


### Verify publish attestation manually

To verify the publish attestation and inspect it, you can use cosign. Install [cosign](https://github.com/lmrs2/bh-aisec/blob/main/INSTALLATION.md#cosign).

Make sure you have access to your image by authenticating to docker:

```shell
$ REGISTRY_TOKEN=<your-token>
$ REGISTRY_USERNAME=<registry-username>
$ docker login -u "${REGISTRY_USERNAME}" "${REGISTRY_TOKEN}"
```

To verify a publish attestation, use the following command:

```shell
# Update the image as recorded in your logs - This is the same as in Activity 01
$ image=docker.io/lmrs2/bh-aisec-model-inference@sha256:b694e56c2c4c60a3e771d7dde95e26c282e1bd6a30a7e8a46c6dc986c54be906
# Update the repository name storing your policies.
$ creator_id="https://github.com/lmrs2/bh-aisec-organization/.github/workflows/image-publisher.yml@refs/heads/main"
$ type=https://slsa.dev/publish/v0.1
$ path/to/cosign verify-attestation "${image}" \
    --certificate-oidc-issuer https://token.actions.githubusercontent.com \
    --certificate-identity "${creator_id}" 
    --type "${type}" | jq -r '.payload' | base64 -d | jq
```

## Create deployment attestation for the model

We will deploy the model in the same service account as in [Activity 03](https://github.com/lmrs2/bh-aisec/tree/main/activities/03), thus we simply need to:

1. Update the list of [packages](https://github.com/lmrs2/bh-aisec-organization/blob/main/policies/deployment/servers-prod.json#L19) using the model container we published in the previous section.

##### Call the evaluator in CI

Follow these steps:

1. Update the [organization workflow call](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/deploy-model.yml#L41) that evaluates the deployment policy.
1. Update the [registry-username](https://github.com/lmrs2/bh-aisec-model/blob/main/.github/workflows/deploy-model.yml#L47) to yours.
1. Run the workflow via the [GitHub UI](https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow#running-a-workflow). It will take ~40s to complete. If all goes well, the workflow run will display a green icon.

##### Verify deployment attestation manually

To verify the publish attestation and inspect it, you can use cosign. Install [cosign](https://github.com/lmrs2/bh-aisec/blob/main/INSTALLATION.md#cosign).

Make sure you have access to your image by authenticating to docker:

```shell
$ REGISTRY_TOKEN=<your-token>
$ REGISTRY_USERNAME=<registry-username>
$ docker login -u "${REGISTRY_USERNAME}" "${REGISTRY_TOKEN}"
```

To verify a deployment attestation, use the following command:

```shell
# Update the image as recorded in your logs
$ image=docker.io/lmrs2/bh-aisec-model-inference@sha256:b694e56c2c4c60a3e771d7dde95e26c282e1bd6a30a7e8a46c6dc986c54be906
# Update the repository name storing your policies.
$ creator_id="https://github.com/lmrs2/bh-aisec-organization/.github/workflows/image-deployer.yml@refs/heads/main"
$ type=https://slsa.dev/deployment/v0.1
$ path/to/cosign verify-attestation "${image}" \
    --certificate-oidc-issuer https://token.actions.githubusercontent.com \
    --certificate-identity "${creator_id}" 
    --type "${type}" | jq -r '.payload' | base64 -d | jq
```

## Make the model available to the cluster

### Download the model
Now that we have verified that our model container is ready for deployment, we can extract the actual model file. We simulate what the control plane would do before making the model available to the inference server. 

```shell
# This is where we will store our model
mkdir my_local_data
# Update with your model
MODEL_IMAGE="docker.io/lmrs2/bh-aisec-model-inference@sha256:b694e56c2c4c60a3e771d7dde95e26c282e1bd6a30a7e8a46c6dc986c54be906"
docker create --name tmp_container $MODEL_IMAGE
docker cp tmp_container:/mnist_classifier.pth my_local_data/mnist_classifier.pth
docker rm tmp_container
```

### Mount the model in the cluster

We assume minikube and kyverno are running, as per [Activity 04](https://github.com/laurentsimon/bh-aisec/tree/main/activities/04).

Run the following command (it if blocks the terminal, add `&` to the command):

```shell
$ minikube mount "$(pwd)/my_local_data:/mnt/local_data"
```

You can verify that the mount is successful using:

```shell
$ minikube ssh
$ ls -l /mnt/local_data/
mnist_classifier.pth
$ exit
```

### Start our deployment

To start the deployment:

1. Update the image in [bh-aisec-project1/k8/echo-server-deployment_predict.yml](https://github.com/lmrs2/bh-aisec-project1/blob/main/k8/echo-server-deployment_predict.yml#L27)
1. Start the deployment and service:

```shell
$ kubectl apply -f bh-aisec-project1/k8/echo-server-deployment_predict.yml 
```

As in [Activity 04](https://github.com/laurentsimon/bh-aisec/tree/main/activities/04), try updating the echo-server image and validate that the policy denies it.

### Test your prediction server

Use images from https://huggingface.co/datasets/ylecun/mnist.

```shell
$ python bh-aisec-project1/images/echo-server/client_repdict.py ./digit4.png http://localhost:8081/classify/v0
```
