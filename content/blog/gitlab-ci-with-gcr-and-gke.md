---
title: "Using Gitlab-CI with GCR and GKE"
date: 2018-11-17T12:00:00-05:00
draft: false
---

A key part of any CI/CD pipeline is getting your application from the build stage
into your staging and production environments. We're going to explore how to
approach this problem using Gitlab-CI, Google Container Registry and Google
Kubernetes Engine.

Let's say you have an application living on Gitlab, and you want to release an image
on Google Container Registry, and deploy it to your Kubernetes cluster automagically.

Thankfully, not just anyone can deploy containers on your cluster, so we'll have
to create a Google Cloud Platform service account with the appropriate roles.

- Go to the [Service Accounts][1] page in your Google Cloud Project
- Create a new service account, with the following roles:
    - Kubernetes Admin
    - Storage Admin
- Generate a JSON key for that account and save it somewhere convenient
- in a shell, run:
    - `$ cat <YOUR_JSON_KEY_FILE> | base64 | xclip -sel clip`

The last command will copy your JSON key to the clipboard after encoding it to
base64. This is not a substitute for good security practices, and would only foil
the casual snoop. Make sure you never expose the key, and treat it as if it were
plain text, because it might as well be. Follow [this conversation on Gitlab][3]
for more info.

Now that we've copied our key, we need to store it as a CI/CD variable in Gitlab,
and create some additional ones to configure the runner with our project and cluster
information. Open up your project's CI/CD settings and expand the Variables section,
and add the followind entries:

- `GCP_SERVICE_KEY`: paste your base64 encoded key from the previous step
- `GCP_PROJECT_NAME`: the name of your google cloud project
- `GCR_REPO`: the full name of your container image (ie. gcr.io/foo/bar)
- `GKE_CLUSTER_ZONE`: the zone where your Kubernetes controller resides
- `GKE_CLUSTER_NAME`: the name of your GKE cluster

If you have a regional cluster, replace `GKE_CLUSTER_ZONE` with `GKE_CLUSTER_REGION`,
and use your Google Cloud cluster region as the value.

Once you have the variables set up, it's time to put them to work and write your
`.gitlab-ci.yml` file. First we'll define a container to build our image using
[Docker-in-Docker][2] Here's what that looks like:

```
build:
    stage: build
    image: docker:latest
    before_script:
        - docker login -u _json_key -p "$(echo $GCP_SERVICE_KEY | base64 -d)" https://gcr.io
    script:
        - docker build --pull -t ${GCR_REPO}:${CI_BUILD_REF} -f Dockerfile .
        - docker push ${GCR_REPO}:${CI_BUILD_REF}
    services:
        - docker:dind
    when: manual
```

Next, we'll spin up a debian runner to deploy our image to our GKE cluster using
the following code:

```
deploy:
    stage: deploy
    before_script:
        - apt-get update && apt-get install apt-transport-https
        - curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg |
          apt-key add -
        - echo "deb https://packages.cloud.google.com/apt cloud-sdk-stretch main" |
          tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
        - echo "deb https://apt.kubernetes.io/ kubernetes-stretch main" |
          tee -a /etc/apt/sources.list.d/kubernetes.list
        - apt-get update &&  apt-get install kubectl google-cloud-sdk -y
        - echo "$(echo $GCP_SERVICE_KEY | base64 -di)" > keyfile
        - gcloud --quiet auth activate-service-account --key-file keyfile
        - gcloud --quiet config set project ${GCP_PROJECT_NAME}
        - gcloud --quiet config set container/cluster ${GKE_CLUSTER_NAME}
        - gcloud --quiet config set compute/zone ${GKE_CLUSTER_ZONE}
        - gcloud --quiet container clusters get-credentials ${GKE_CLUSTER_NAME}
    script:
        - kubectl apply -f deployment.yaml
    when: manual
```

If you have a regional cluster, replace:

```
- gcloud --quiet config set compute/zone ${GKE_CLUSTER_ZONE}
```

with:

```
- gcloud --quiet config set compute/region ${GKE_CLUSTER_REGION}
```

The first half of the before_script installs `google-cloud-sdk` and `kubectl`,
then decodes the service key into a file and use it to authenticate with Google.
The next part configures your project and cluster parameters, and the last line
generates your `.kube/config`.

How you choose to deploy your app is up to you. You can use `kubectl apply`,
like the example above, but you'd be better off using [Helm][4]. If you want to
install the Helm client, add the following to your before_script:

```
- wget https://raw.githubusercontent.com/helm/helm/master/scripts/get -O get_helm.sh
- chmod +x get_helm.sh
- ./get_helm.sh
- helm init --client-only
```

If you want to avoid installing `google-cloud-sdk` on your Gitlab runner,
you can instead generate the kubeconfig on your own machine using the same gcloud
commands, save your (encoded) `./kube/config` to a CI/CD variable, and decode it
into the `$KUBECONFIG` variable, which Gitlab reserves for your kubernetes
configuration. Or you can forget about the base64 encoding entirely and load it
straight into `$KUBECONFIG`.

Hopefully this has saved you some pain, or better yet, given you a useful tool
to automate your CI/CD strategy. Come back soon for more!

[1]: https://console.cloud.google.com/iam-admin/serviceaccounts
[2]: https://hub.docker.com/_/docker/
[3]: https://gitlab.com/gitlab-org/gitlab-ce/issues/13784
[4]: https://helm.sh
