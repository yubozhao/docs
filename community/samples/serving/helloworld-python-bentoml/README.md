---
title: "Hello World - Python BentoML"
linkTitle: "Python Bentoml"
weight: 1
type: "docs"
---

A simple machine learning model with API serving that is written in python and
using [BentoML](https://github.com/bentoml/BentoML).

This sample will walk you through the steps of creating and deploying a machine learning
model using python. It will use BentoML to package a classifier model trained
on the Iris dataset. Afterward we will create an API model server container image and
deploy the image to Knative.

Additional Knative deployment guide with BentoML is also available in the
[BentoML documentation](https://docs.bentoml.org/en/latest/deployment/knative.html)

## Before you begin

- A Kubernetes cluster with Knative installed. Follow the
  [installation instructions](../../../../docs/install/README.md) if you need to
  create one.
- [Docker](https://www.docker.com) installed and running on your local machine,
  and a Docker Hub account configured (we'll use it for a container registry).
- Python 3.6 or above installed and running on your local machine.
  - `scikit-learn` and `bentoml` packages, run command:

      ```shell
      pip install scikit-learn
      pip install bentoml
      ```

## Recreating sample code

1. Save the following code into file called `iris_classifier.py`. This code defines a
  machine learning service that requires a scikit-learn model with BentoML. It asks
  BentoML to figure out the required pip dependencies, also defined an API which is the
  entry point for accessing this machine learning service:

    ```python
    from bentoml import env, artifacts, api, BentoService
    from bentoml.handlers import DataframeHandler
    from bentoml.artifact import SklearnModelArtifact

    @env(auto_pip_dependencies=True)
    @artifacts([SklearnModelArtifact('model')])
    class IrisClassifier(BentoService):

        @api(DataframeHandler)
        def predict(self, df):
            return self.artifacts.model.predict(df)
    ```

2. Save the following code into file called `helloworld_bentoml.py`. This code defines
  how to train classifier model with iris dataset and how to save the model with
  BentoML.

    ```python
    from sklearn import svm
    from sklearn import datasets

    # import the class from the file we created from previous step
    from iris_classifier import IrisCLassifier

    if __name__ == "__main__":
        # Load training data
        iris = datasets.load_iris()
        X, y = iris.data, iris.target

        # Model Training
        clf = svm.SVC(gamma='scale')
        clf.fit(X, y)

        # Create a iris classifier service instance
        iris_classifier_service = IrisClassifier()

        # Pack the newly trained model artifact
        iris_classifier_service.pack('model', clf)

        # Save the prediction service to disk for model serving
        saved_path = iris_classifier_service.save()
    ```

3. Run following command to execute the file we wrote, it will trains the model and
  save the model with its dependencies and configuration in a versioned file archive on
  your local disk with BentoML:

    ```shell
    python helloworld_bentoml.py
    ```

4. We can use BentoML CLI to get saved model info.

    ```shell
    bentoml get IrisClassifier:latest
    ```

    Example:

    ```shell
    > bentoml get IrisClassifier:latest
    {
      "name": "IrisClassifier",
      "version": "20200305171229_0A1411",
      "uri": {
        "type": "LOCAL",
        "uri": "/Users/bozhaoyu/bentoml/repository/IrisClassifier/20200305171229_0A1411"
      },
      "bentoServiceMetadata": {
        "name": "IrisClassifier",
        "version": "20200305171229_0A1411",
        "createdAt": "2020-03-06T01:12:49.431011Z",
        "env": {
          "condaEnv": "name: bentoml-IrisClassifier\nchannels:\n- defaults\ndependencies:\n- python=3.7.3\n- pip\n",
          "pipDependencies": "bentoml==0.6.2\nscikit-learn",
          "pythonVersion": "3.7.3"
        },
        "artifacts": [
          {
            "name": "model",
            "artifactType": "SklearnModelArtifact"
          }
        ],
        "apis": [
          {
            "name": "predict",
            "handlerType": "DataframeHandler",
            "docs": "BentoService API",
            "handlerConfig": {
              "orient": "records",
              "typ": "frame",
              "input_dtypes": null,
              "output_orient": "records"
            }
          }
        ]
      }
    }
    ```

### Test run API server

BentoML can start an API server from the saved model. We will use BentoML CLI command to
start an API server locally and test it with curl command.

To start the API server use following command:

  ```shell
  bentoml serve IrisClassifier:latest
  ```

In another terminal window, we can make `curl` request with sample data to the API server
and get prediction result:

  ```shell
  curl -v -i \
  --header "Content-Type: application/json" \
  --request POST \
  --data '[[5.1, 3.5, 1.4, 0.2]]' \
  127.0.0.1:5000/predict
  ```

## Building and deploying the sample

BentoML auto generates a dockerfile for API server of the saved model.

1. Use Docker to build API server into docker image and push with Docker hub. Run these
  commands replacing `{username}` with your Docker hub username.

    ```shell
    saved_path=$(bentoml get IrisClassifier:latest -q | jq -r ".uri.uri")
    # Build the container on your local machine
    docker build - t {username}/iris-classifier $saved_path

    # Push the container to docker registry
    docker push {username}/iris-classifier
    ```

    Example:

    ```shell
    saved_path=$(bentoml get IrisClassifier:latest -q | jq -r ".uri.uri")
    docker build -t yubozhao/iris-classifier $saved_path

    docker push yubozhao/iris-classifier
    ```

2. After the build has completed the container is pushed to the docker
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

3. Run the following command to find the domain URL for your service:

  ```shell
  kubectl get ksvc iris-classifier --output=custom-columns=NAME:.metadata.name,URL:.status.url
  ```

  Example:

  ```shell
  > kubectl get ksvc iris-classifier --output=custom-columns=NAME:.metadata.name,URL:.status.url

  NAME            URL
  iris-classifer   http://iris-classifer.default.example.com
  ```

4. Now you can make a request to your app and see the result. Replace
  the URL below with the URL returned in the previous command.

  ```shell
  curl -v -i \
    --header "Content-Type: application/json" \
    --request POST \
    --data '[[5.1, 3.5, 1.4, 0.2]]' \
    http://iris-classifier.default.example.com/predict
  ```

  Example:

  ```shell
  > curl -v -i \
    --header "Content-Type: application/json" \
    --request POST \
    --data '[[5.1, 3.5, 1.4, 0.2]]' \
    http://iris-classifier.default.example.com/predict
  [0]
  ```

## Removing the sample app deployment

To remove the application from your cluster, delete the service record:

  ```shell
  kubectl delete --filename service.yaml
  ```
