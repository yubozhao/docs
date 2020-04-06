---
title: "Iris Classifier - serving"
linkTitle: "Machine learning serving"
weight: 1
type: "docs"
---

A simple REST API machine learning application that is written in python and using [BentoML](https://github.com/bentoml/BentoML).


This sample will walk you through the steps of training, creating and deploying an iris
classifier using python. It will use BentoML to package and bundle the iris classifier
model and then use docker to build and push container image to docker hub, and finally
deploying your ML app to your Knative cluster.


## Before you begin

- A Kubernetes cluster with Knative installed. Follow the
  [installation instructions](../../../../docs/install/README.md) if you need to
  create one.
- [Docker](https://www.docker.com) installed and running on your local machine,
  and a Docker Hub account configured (we'll use it for a container registry).
- Python 3.6 or above installed and running on your local machine.
  - `scikit-learn` and `bentoml` packages, run command:

      ```shell
      pip install scikit-learn bentoml
      ```

## Creating application

1. Save the following code into file called `iris_classifier.py`:

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

2. Save the following  code into file called `helloworld_ml.py`:
    ```python
    from sklearn import svm
    from sklearn import datasets

    # import the class we create from previous step
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

3. Run following command to build machine learing model and then bundle it with BentoML:
    ```shell
    python helloworld_ml.py
    ```

Now you have a packaged machine learning model that is ready for deloyment.


## Building application

1. Use BentoML CLI to get packaged model information

    ```shell
    bentoml get IrisClassifier:latest
    ```

    Example output:
    ```shell
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

2. Use Docker to build the application into a container. To build and
  push with Docker hub, run these commands replacing `{username}`
  with your Docker hub username and replacing `{model path}` from the `uri.uri` section from the output above:

    ```shell
    # Build the container on your local machine
    docker build - t {username}/iris-classifier {model_path}

    # Push the container to docker registry
    docker push {username}/iris-classifier
    ```

    Example:
    ```shell
    docker build -t yubozhao/iris-classifier /Users/bozhaoyu/bentoml/repository/IrisClassifier/20200305171229_0A1411

    docker push yubozhao/iris-classifier
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
  kubectl get ksvc iris-classifier --output=custom-columns=NAME:.metadata.name,URL:.status.url
  ```

  Examplei ouput:
  ```shell
  NAME            URL
  iris-classifer   http://iris-classifer.default.example.com
  ```

## Making request to service

1. Now you can make a request to your app and see the result. Replace
  the URL below with the URL returned in the previous command.

  ```shell
  curl -v -i \
    --header "Content-Type: application/json" \
    --request POST \
    --data '[[5.1, 3.5, 1.4, 0.2]]' \
    http://iris-classifier.default.example.com/predict
  ```

  Example output:
  ```
  [0]
  ```


## Removing deployment

To remove the application from your cluster, delete the service record:

  ```shell
  kubectl delete --filename service.yaml
  ```
