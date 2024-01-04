---
description: Creating a full picture of a ML model using the Model Control Plane
---

# Keeping track of ML models in ZenML

![Walkthrough of ZenML Model Control Plane (Dashboard available only on ZenML Cloud)](../../.gitbook/assets/mcp_walkthrough.gif)

As discussed in the [Core Concepts](../../getting-started/core-concepts.md), ZenML also contains the notion of a `Model`, which consists of many `ModelVersions` (the iterations of the model). These concepts are exposed in the `Model Control Plane` (MCP for short).

## What is a ZenML Model?

Before diving in, let's take some time to build an understanding of what we mean when we say `Model` in ZenML terms. A `Model` is simply an entity that groups pipelines, artifacts, metadata, and other crucial business data into a unified entity. Please note that one of the most common artifacts that is associated with a Model in ZenML is the so-called technical model, which is the actual model file/files that hold the weight and parameters of a machine learning training result. However, this is not the only artifact that is relevant; artifacts such as the training data and the predictions this model produces in production are also linked inside a ZenML Model. In this sense, a ZenML Model is a concept that more broadly encapsulates your ML product's business logic.

Models are first-class citizens in ZenML and as such
viewing and using them is unified and centralized in the ZenML API, the ZenML client as
well as on the [ZenML Cloud](https://zenml.io/cloud) dashboard.

These models can be viewed within ZenML:

{% tabs %}
{% tab title="OSS (CLI)" %}

`zenml model list` can be used to list all models.

{% endtab %}
{% tab title="Cloud (Dashboard)" %}

The [ZenML Cloud](https://zenml.io/cloud) dashboard has additional capabilities, that include visualizing these models in the dashboard.

<figure><img src="../../.gitbook/assets/mcp_model_list.png" alt=""><figcaption><p>ZenML Model Control Plane.</p></figcaption></figure>

{% endtab %}
{% endtabs %}

## Configuring a model in a pipeline

The easiest way to use a ZenML model is to pass a model version object as part of a pipeline run. This can be done easily at a pipeline or a step level, or via a 
[YAML config](../production-guide/configure-pipeline.md).

Once you configure a pipeline this way, **all** artifacts generated during pipeline runs are automatically linked to the specified model version. This connecting of artifacts provides lineage tracking and transparency into what data and models are used during training, evaluation, and inference.

```python
from zenml import pipeline
from zenml.model import ModelVersion

model_version = ModelVersion(
    # The name uniquely identifies this model
    # It usually represents the business use case
    name="iris_classifier",
    # The version specifies the version
    # If None or an unseen version is specified, it will be created
    # Otherwise, a version will be fetched.
    version=None, 
    # Some other properties may be specified
    license="Apache 2.0",
    description="A classification model for the iris dataset.",
)

# The step configuration will take precedence over the pipeline
@step(model_version=model_version)
def svc_trainer(...) -> ...:
    ...

# This configures it for all steps within the pipeline
@pipeline(model_version=model_version)
def training_pipeline(gamma: float = 0.002):
    # Now this pipeline will have the `iris_classifier` model active.
    X_train, X_test, y_train, y_test = training_data_loader()
    svc_trainer(gamma=gamma, X_train=X_train, y_train=y_train)

if __name__ == "__main__":
    training_pipeline()

# In the YAML the same can be done; in this case, the 
#  passing to the decorators is not needed
# model_version: 
  # name: iris_classifier
  # license: "Apache 2.0"
  # description: "A classification model for the iris dataset."

```

The above will establish a **link between all artifacts that pass through this ZenML pipeline and this model**. This includes the **technical model** which is what comes out of the `svc_trainer` step. You will be able to see all associated artifacts and pipeline runs, all within one view.

Further, this pipeline run and all other pipeline runs that are configured with this model version will be linked to this model as well. 

You can see all versions of a model, and associated artifacts and run like this:

{% tabs %}
{% tab title="OSS (CLI)" %}

`zenml model version list <MODEL_NAME>` can be used to list all versions of a particular model.

The following commands can be used to list the various pipeline runs associated with a model:

* `zenml model version runs <MODEL_NAME> <MODEL_VERSIONNAME>`

The following commands can be used to list the various artifacts associated with a model:

* `zenml model version data_artifacts <MODEL_NAME> <MODEL_VERSIONNAME>`
* `zenml model version model_artifacts <MODEL_NAME> <MODEL_VERSIONNAME>`
* `zenml model version deployment_artifacts <MODEL_NAME> <MODEL_VERSIONNAME>`

{% endtab %}
{% tab title="Cloud (Dashboard)" %}

The [ZenML Cloud](https://zenml.io/cloud) dashboard has additional capabilities, that include visualizing all associated runs and artifacts for a model version:

<figure><img src="../../.gitbook/assets/mcp_model_versions_list.png" alt="ZenML Model Versions List."><figcaption><p>ZenML Model Versions List.</p></figcaption></figure>

{% endtab %}
{% endtabs %}

## Fetching the model in a pipeline

When configured at the pipeline or step level, the model version will be available through the [StepContext](../advanced-guide/pipelining-features/fetch-metadata-within-pipeline.md) or [PipelineContext](../advanced-guide/pipelining-features/fetch-metadata-within-pipeline.md).

```python
from zenml import get_step_context, get_pipeline_context, step, pipeline

@step
def svc_trainer(
    X_train: pd.DataFrame,
    y_train: pd.Series,
    gamma: float = 0.001,
) -> Annotated[ClassifierMixin, "trained_model"]:
    # This will return the model version specified in the 
    # @pipeline decorator. In this case, the production version of 
    # the `iris_classifier` will be returned in this case.
    model_version = get_step_context().model_version
    ...

@pipeline(
    model_version=ModelVersion(
        # The name uniquely identifies this model
        name="iris_classifier",
        # Pass the stage you want to get the right model
        version="production", 
    ),
)
def training_pipeline(gamma: float = 0.002):
    # Now this pipeline will have the production `iris_classifier` model active.
    model_version = get_pipeline_context().model_version

    X_train, X_test, y_train, y_test = training_data_loader()
    svc_trainer(gamma=gamma, X_train=X_train, y_train=y_train)
```

## Logging metadata to the `ModelVersion` object

[Just as one can associate metadata with artifacts](manage-artifacts.md#logging-metadata-for-an-artifact), model versions too can take a dictionary
of key-value pairs to capture their metadata. This is achieved using the 
`log_model_metadata` method:

```python
from zenml import get_step_context, step, log_model_metadata 

@step
def svc_trainer(
    X_train: pd.DataFrame,
    y_train: pd.Series,
    gamma: float = 0.001,
) -> Annotated[ClassifierMixin, "sklearn_classifier"],:
    # Train and score model
    ...
    model.fit(dataset[0], dataset[1])
    accuracy = model.score(dataset[0], dataset[1])

    model_version = get_step_context().model_version
    
    log_model_metadata(
        # Model name can be omitted if specified in the step or pipeline context
        model_name="iris_classifier",
        # Passing None or omitting this will use the `latest` version
        version=None,
        # Metadata should be a dictionary of JSON-serializable values
        metadata={"accuracy": float(accuracy)}
        # A dictionary of dictionaries can also be passed to group metadata
        #  in the dashboard
        # metadata = {"metrics": {"accuracy": accuracy}}
    )
```

{% tabs %}
{% tab title="Python" %}

```python
from zenml.client import Client

# Get an artifact version (in this the latest `iris_classifier`)
model_version = Client().get_model_version('iris_classifier')

# Fetch it's metadata
model_version.run_metadata["accuracy"].value
```

{% endtab %}
{% tab title="Cloud (Dashboard)" %}

The [ZenML Cloud](https://zenml.io/cloud) dashboard offers advanced visualization features for artifact exploration, including a dedicated artifacts tab with metadata visualization:

<figure><img src="../../.gitbook/assets/dcp_metadata.png" alt=""><figcaption><p>ZenML Artifact Control Plane.</p></figcaption></figure>

{% endtab %}
{% endtabs %}

Choosing [log metadata with artifacts](manage-artifacts.md#logging-metadata-for-an-artifact) or model versions depends on the scope and purpose of the information you wish to capture. Artifact metadata is best for details specific to individual outputs, while model version metadata is suitable for broader information relevant to the overall model. By utilizing ZenML's metadata logging capabilities and special types, you can enhance the traceability, reproducibility, and analysis of your ML workflows.

For further depth, there is an [advanced metadata logging guide](../advanced-guide/data-management/logging-metadata.md) that goes more into detail about logging metadata in ZenML.

## Using the stages of a model

A model's versions can exist in various stages. These are meant to signify their lifecycle state:

* `staging`: This version is staged for production.
* `production`: This version is running in a production setting.
* `latest`: The latest version of the model.
* `archived`: This is archived and no longer relevant. This stage occurs when a model moves out of any other stage.

{% tabs %}
{% tab title="Python SDK" %}
```python
from zenml.model import ModelVersion

# Get the latest model version
model_version = ModelVersion(
    name="iris_classifier",
    version="latest"
)

# Get a model from a version
model_version = ModelVersion(
    name="iris_classifier",
    version="my_version",
)

# Pass the stage into the version field
# to get the model by stage
model_version = ModelVersion(
    name="iris_classifier",
    version="staging",
)

# This will set this version to production
model_version.set_stage(stage="production", force=True)
```
{% endtab %}

{% tab title="CLI" %}
```shell
# List staging models
zenml model version list <MODEL_NAME> --stage staging 

# Update to production
zenml model version update <MODEL_NAME> <MODEL_VERSIONNAME> -s production 
```
{% endtab %}
{% tab title="Cloud (Dashboard)" %}
The [ZenML Cloud](https://zenml.io/cloud) dashboard has additional capabilities, that include easily changing the stage:

![ZenML Cloud Transition Model Stages](../../.gitbook/assets/dcp_transition_stage.gif)

{% endtab %}
{% endtabs %}

## Facilitating artifacts exchange between pipelines using MCP

A ZenML Model spans multiple pipelines and is a key concept that brings disparate pipelines together. A simple example is illustrated below:

<figure><img src="../../.gitbook/assets/mcp_pipeline_overview.png" alt=""><figcaption><p>A simple example of two pipelines interacting between each other.</p></figcaption></figure>

Each time the `train_and_promote` pipeline runs, it creates a new `iris_classifier`. However, it only promotes the created model to `production` if a certain accuracy threshold is met. The `do_predictions` pipeline simply picks up the latest promoted model and runs batch inference on it. That way these two pipelines can independently be run, but can rely on each other's output.

One way of achieving this is to fetch the model directly in your step:

```python
from zenml import step, get_step_context

# IMPORTANT: Cache needs to be disabled to avoid unexpected behavior
@step(enable_cache=False)
def predict(
    data: pd.DataFrame,
) -> Annotated[pd.Series, "predictions"]:
    # model_name and model_version derived from pipeline context
    model_version = get_step_context().model_version

    # Fetch the model directly from the model control plane
    model = model_version.get_model_artifact("trained_model")

    # Make predictions
    predictions = pd.Series(model.predict(data))
    return predictions
```

However, this approach has the downside that if the step is cached, then it could lead to unexpected results. You could simply disable the cache in the above step or the corresponding pipeline. However, one other way of achieving this would be to resolve the artifact at the pipeline level:

```python
from typing_extensions import Annotated
from zenml import get_pipeline_context, pipeline, ExternalArtifact
from zenml.enums import ModelStages
from zenml.model import ModelVersion
import pandas as pd
from sklearn.base import ClassifierMixin


@step
def predict(
    model: ClassifierMixin,
    data: pd.DataFrame,
) -> Annotated[pd.Series, "predictions"]:
    predictions = pd.Series(model.predict(data))
    return predictions

@pipeline(
    model_config=ModelVersion(
        name="iris_classifier",
        # Using the production stage
        version=ModelStages.PRODUCTION,
    ),
)
def do_predictions():
    # model_name and model_version derived from pipeline context
    model_version = get_pipeline_context().model_version
    inference_data = load_data()
    predict(
        # Here, we load in the `trained_model` from a trainer step
        model=model_version.get_model_artifact("trained_model"),  
        data=inference_data,
    )


if __name__ == "__main__":
    do_predictions()
```

Ultimately, both approaches are fine. Users should decide which one to use based on their own preferences.

ZenML Model and Model Versions are some of the most powerful features in ZenML. To understand them in a deeper way, read the [dedicated Model Management](../advanced-guide/data-management/model-management.md).
guide.

<!-- For scarf -->
<figure><img alt="ZenML Scarf" referrerpolicy="no-referrer-when-downgrade" src="https://static.scarf.sh/a.png?x-pxid=f0b4f458-0a54-4fcd-aa95-d5ee424815bc" /></figure>