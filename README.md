# GCP Container Builder with PHP and Composer

This repo is an example of using [Cloud Container Builder](https://cloud.google.com/container-builder/docs/quickstart-gcloud) for automatic builds of a PHP-FPM/Composer/Nginx site hosted in Google Container Engine.

This repo was inspired by the talk [Container management and deployment: from development to production](https://www.youtube.com/watch?v=XL9CQobFB8I) given at [Google Cloud Next '17](https://www.youtube.com/playlist?list=PLIivdWyY5sqI8RuUibiH8sMb1ExIw0lAR) by [Kelsey Hightower](https://twitter.com/kelseyhightower).

[![Container management and deployment: from development to production (Google Cloud Next '17)](https://img.youtube.com/vi/XL9CQobFB8I/0.jpg)](https://www.youtube.com/watch?v=XL9CQobFB8I)

## Helm

This repo also has a Helm branch, with an example of deploying using charts:

[Deploying using charts](https://github.com/Simon-Ince/gcp-container-builder-with-php-composer/tree/helm).

## Prerequisites

* A [GCP account](https://cloud.google.com/) with a [Project](https://cloud.google.com/resource-manager/docs/creating-managing-projects) already established.

* [A GCP container clusters created](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create) called `production	`.

* [Google Cloud SDK installed locally](https://cloud.google.com/sdk/downloads), and [logged in](https://cloud.google.com/sdk/gcloud/reference/auth/login).

## Setup

* Fork this repo and pull locally.

* Change the environment variables in `bin/create-production-kubeconfig` and run `. ./bin/create-production-kubeconfig` from the repo's root.

* Change the image names in `production/container-builder-test-deploy.yaml`, swapping the project names for your own.

* [Create a Build Trigger for the forked repo](https://cloud.google.com/container-builder/docs/creating-build-triggers) in the GCP dashboard.

You're all done, pushing to your forked Git repo will trigger new builds.

## What's going on here?

GCP Container Builder will read through the steps in `cloudbuild.yaml` when a push is made to Github.

1.
```
- name: 'composer/composer'
  args: [ 'install' ]
  dir: "php"
  id: "php-composer"
```
The first step pulls the composer/composer image from Docker Hub and runs `install` in the `./php` directory, building out any dependancies.

2.
```
- name: 'gcr.io/cloud-builders/docker'
  args: [ 'build', '-f', 'Dockerfile-php', '-t', 'gcr.io/$PROJECT_ID/php', '.' ]
  id: "docker-build-php"
  waitFor:
  - "php-composer"
```
Next, the `gcr.io/cloud-builders/docker` container is used to build the `Dockerfile-php` file, which in itself pull in the `./app` directory from the previous step.

3.
```
- name: 'gcr.io/cloud-builders/docker'
  args: [ 'build', '-f', 'Dockerfile-nginx', '-t', 'gcr.io/$PROJECT_ID/nginx', '.' ]
  id: "docker-build-nginx"
```
This step is repeated for the `Dockerfile-nginx`, plus `nginx/site.conf` is added to our Nginx container with our config.

4.
```
- name: gcr.io/cloud-builders/gsutil
  args: ["cp", "gs://${PROJECT_ID}/production.kubeconfig", "/workspace/kubeconfig"]
```
The `gcr.io/cloud-builders/gsutil` image is used to access the GCP Bucket we uploaded our production.kubeconfig to earlier, moving it to our workspace.

5.
```
- name: "gcr.io/google_containers/hyperkube:v1.5.3"
  env: ["KUBECONFIG=/workspace/kubeconfig"]
  entrypoint: "/hyperkube"
  args: ["kubectl", "apply", "-f", "/workspace/production"]
```
The `gcr.io/google_containers/hyperkube:v1.5.3` image uses `production.kubeconfig` to access our cluster and deploy our Kubernetes manifests.

6.
```
images:
- 'gcr.io/$PROJECT_ID/php'
- 'gcr.io/$PROJECT_ID/nginx'
```
Our newly built image are push to our GCP Container Registry.

## Todo

* Currently, images are pushed to the Registry after the deployment steps have been completed, which means the images from the previous and not current build are being deployed. I have attempted to move the `images` map further up but it breaks the `steps` map.

* ~~Create a Helm branch so less manual intervention is required.~~ Done.
