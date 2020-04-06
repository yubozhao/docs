---
title: "Fashion Mnist - serving"
linkTitle: "Machine learning serving"
weight: 1
type: "docs"
---


A simple machine learning application that predict in real time with BentoML.
BentoML creats a REST API.


Follow the steps below to create the sample application and then deploy the app
to your cluster.


## Before you begin

- A Kubernetes cluster with Knative installed. Follow the
  [installation instructions](../../../../docs/install/README.md) if you need to
  create one.
- [Docker](https://www.docker.com) installed and running on your local machine,
  and a Docker Hub account configured (we'll use it for a container registry).
- Python 3.6 or above installed and running on your local machine.

## Creating application

1. Using jupter notebook to run `fashion_mnist.ipynb`, paste the following code

    ```bash
    jupyter notebook ./tensorflow_fashion_mnist.ipynb
    ```

2. In browser, click run all cells in the cell section of navigation bar. You should see packaged model information in last cell's output.

## Building application

1. Use BentoML CLI to get packaged model information

    ```bash
    bentoml get fashion-mnist:latest
    ```

2. Use Docker to build the application into a container. To build and
  push with Docker hub, run these commands replacing `{username}`
  with your Docker hub username:

  ```shell
  # Build the container on your local machine
  cd {model_path}
  docker build - t {username}/fashion-mnist {model_path}

  # Push the container to docker registry
  docker push {username}/fashion-mnist
  ```

## Deploying application

1. After the build has completed the container is pushed to the docker
  hub, you can deploy the application into your cluster. Ensure that
  the container image value in `service.yaml` matches the container you
  built in the previous step. Apply the configuration use `kubectl`:

  ```shell
  kubectl apply --filename service.yaml
  ```

Now that your service is created, Knative performs the following steps:

  - Create a new immutable revision for this version of the app.
  - Network programming to create a route, ingress, service and load
    balance for your application.
  - Automatically scale your pods up and down (including to zero active
    pods).

2. Run the following command to find the domain URL for your service:

  ```shell
  kubectl get ksvc fashion-mnist --output=custom-columns=NAME:.metadata.name,URL:.status.url
  ```

  Example:
  ```shell
  NAME            URL
  fashion-mnist   http://fashion-mnist.default.example.com
  ```

## Making request to service

1. Now you can make a request to your app and see the result. Replace
  the URL below with the URL returned in the previous command.

  ```shell
  curl -v -i \
    --header "Content-Type: application/json" \
    --request POST \
    --data-binary @test.json \
    http://fashion-mnist.default.example.com/predict
  ```

  Example:
  ```shell
  [1] result
  ```

## Removing deployment

To remove the application from your cluster, delete the service record:

  ```shell
  kubectl delete --filename service.yaml
  ```
