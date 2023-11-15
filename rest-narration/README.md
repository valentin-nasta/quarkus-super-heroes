# Superheroes Narration Microservice

## Table of Contents
- [Introduction](#introduction)
    - [Exposed Endpoints](#exposed-endpoints)
- [Contract testing with Pact](#contract-testing-with-pact)
- [Running the Application](#running-the-application)
    - [Integration with OpenAI Providers](#integration-with-openai-providers) 
- [Running Locally via Docker Compose](#running-locally-via-docker-compose)
- [Deploying to Kubernetes](#deploying-to-kubernetes)
    - [Using pre-built images](#using-pre-built-images)
    - [Deploying directly via Kubernetes Extensions](#deploying-directly-via-kubernetes-extensions)

## Introduction
This is the Narration REST API microservice. It is a reactive HTTP microservice using the [Microsoft Semantic Kernel for Java](https://devblogs.microsoft.com/semantic-kernel/introducing-semantic-kernel-for-java/) to integrate with an AI service to generate text narrating a given fight.

The Narration microservice needs to access an AI service to generate the text narrating the fight. You can choose between [OpenAI](https://openai.com/) or [Azure OpenAI](https://azure.microsoft.com/en-us/products/ai-services/openai-service). Azure OpenAI, or "OpenAI on Azure" is a service that provides REST API access to OpenAI’s models, including the GPT-4, GPT-3, Codex and Embeddings series. The difference between OpenAI and Azure OpenAI is that it runs on Azure global infrastructure, which meets your production needs for critical enterprise security, compliance, and regional availability.

This service is implemented using [RESTEasy Reactive](https://quarkus.io/guides/resteasy-reactive) with reactive endpoints. Additionally, this application favors constructor injection of beans over field injection (i.e. `@Inject` annotation).

![rest-narration](images/rest-narration.png)

### Exposed Endpoints
The following table lists the available REST endpoints. The [OpenAPI document](openapi-schema.yml) for the REST endpoints is also available.

| Path                   | HTTP method | Response Status | Response Object | Description                                                                                                                     |
|------------------------|-------------|-----------------|-----------------|---------------------------------------------------------------------------------------------------------------------------------|
| `/api/narration`       | `POST`      | `200`           | `String`        | Creates a narration for the passed in [`Fight`](src/main/java/io/quarkus/sample/superheroes/narration/Fight.java) request body. |
| `/api/narration`       | `POST`      | `400`           |                 | Invalid [`Fight`](src/main/java/io/quarkus/sample/superheroes/narration/Fight.java)                                             |
| `/api/narration/hello` | `GET`       | `200`           | `String`        | Ping "hello" endpoint                                                                                                           |

## Contract testing with Pact
[Pact](https://pact.io) is a code-first tool for testing HTTP and message integrations using `contract tests`. Contract tests assert that inter-application messages conform to a shared understanding that is documented in a contract. Without contract testing, the only way to ensure that applications will work correctly together is by using expensive and brittle integration tests.

[Eric Deandrea](https://developers.redhat.com/author/eric-deandrea) and [Holly Cummins](https://hollycummins.com) recently spoke about contract testing with Pact and used the Quarkus Superheroes for their demos. [Watch the replay](https://www.youtube.com/watch?v=vYwkDPrzqV8) and [view the slides](https://hollycummins.com/modern-microservices-testing-pitfalls-devoxx/) if you'd like to learn more about contract testing.

The `rest-narration` application is a [Pact _Provider_](https://docs.pact.io/provider), and as such, should run provider verification tests against contracts produced by consumers.

As [this README states](src/test/resources/pacts/README.md), contracts generally should be hosted in a [Pact Broker](https://docs.pact.io/pact_broker) and then automatically discovered in the provider verification tests.

One of the main goals of the Superheroes application is to be super simple and just "work" by anyone who may clone this repo. That being said, we can't make any assumptions about where a Pact broker may be or any of the credentials required to access it.

Therefore, the [Pact contract](src/test/resources/pacts/rest-fights-rest-narration.json) is committed into this application's source tree inside the [`src/test/resources/pacts` directory](src/test/resources/pacts). In a realistic
scenario, if a broker wasn't used, the consumer's CI/CD would commit the contracts into this repository's source control.

The Pact tests use the [Quarkus Pact extension](https://github.com/quarkiverse/quarkus-pact). This extension is recommended to give the best user experience and ensure compatibility.

## Running the Application
The application runs on port `8087` (defined by `quarkus.http.port` in [`application.properties`](src/main/resources/application.properties)).

From the `quarkus-super-heroes/rest-narration` directory, simply run `./mvnw quarkus:dev` to run [Quarkus Dev Mode](https://quarkus.io/guides/maven-tooling#dev-mode), or running `quarkus dev` using the [Quarkus CLI](https://quarkus.io/guides/cli-tooling). The application will be exposed at http://localhost:8087 and the [Quarkus Dev UI](https://quarkus.io/guides/dev-ui) will be exposed at http://localhost:8087/q/dev.

> [!IMPORTANT]
> This app currently does **NOT** support running in native mode due to incompatibilities in the Microsoft Semantic Kernel library and native compilation. Once that is resolved then native support will be enabled for this application.
>
> See https://github.com/microsoft/semantic-kernel/issues/2885 for details.

### Integration with OpenAI Providers
Currently the only supported OpenAI providers are the [Microsoft Azure OpenAI Service](https://azure.microsoft.com/en-us/products/ai-services/openai-service) and [OpenAI](https://openai.com/) using the [Microsoft Semantic Kernel for Java](https://devblogs.microsoft.com/semantic-kernel/introducing-semantic-kernel-for-java/). This integration requires creating resources, either on OpenAI or Azure, in order to work properly.

For Azure, the [`create-azure-openai-resources.sh` script](../scripts/create-azure-openai-resources.sh) can be used to create the required Azure resources. Similarly, the [`delete-azure-openai-resources.sh` script](../scripts/delete-azure-openai-resources.sh) can be used to delete the Azure resources.

> [!CAUTION]
> Using Azure OpenAI or OpenAI may not be a free resource for you, so please understand this! Unless configured otherwise, this application does **NOT** communicate with any external service. Instead, by default, it just returns a default narration.

The skill definition(s) can be found in the [`src/main/resources/skills` directory](src/main/resources/skills).

Because of this integration and our goal to keep this application working at all times, all the OpenAI integration is disabled by default. A default narration will be provided instead.

To enable the OpenAI integration the following properties must be set, either in [`application.properties`](src/main/resources/application.properties) or as environment variables:

#### Azure OpenAI properties

| Description                                                                                    | Environment Variable                              | Java Property                                     | Value                                                                   |
|------------------------------------------------------------------------------------------------|---------------------------------------------------|---------------------------------------------------|-------------------------------------------------------------------------|
| Enable Azure OpenAI integration                                                                | `NARRATION_AZURE_OPEN_AI_ENABLED`                 | `narration.azure-open-ai.enabled`                 | `true`                                                                  |
| Azure cognitive services account key                                                           | `NARRATION_AZURE_OPEN_AI_KEY`                     | `narration.azure-open-ai.key`                     | `Your azure openai key`                                                 |
| The Azure OpenAI endpoint                                                                      | `NARRATION_AZURE_OPEN_AI_ENDPOINT`                | `narration.azure-open-ai.endpoint`                | `Your azure openai endpoint`                                            |
| Azure cognitive services deployment name                                                       | `NARRATION_AZURE_OPEN_AI_DEPLOYMENT_NAME`         | `narration.azure-open-ai.deployment-name`         | `csdeploy-super-heroes`                                                 |
| Any additional properties to be passed to the Microsoft Semantic Kernel `OpenAIClientProvider` | `NARRATION_AZURE_OPEN_AI_ADDITIONAL_PROPERTIES_*` | `narration.azure-open-ai.additional-properties.*` | Any set of key/value pairs to be passed into the `OpenAIClientProvider` |

#### OpenAI properties

| Description                                                                                    | Environment Variable                        | Java Property                               | Value                                                                   |
|------------------------------------------------------------------------------------------------|---------------------------------------------|---------------------------------------------|-------------------------------------------------------------------------|
| Enable OpenAI integration                                                                      | `NARRATION_OPEN_AI_ENABLED`                 | `narration.open-ai.enabled`                 | `true`                                                                  |
| OpenAI API Key                                                                                 | `NARRATION_OPEN_AI_API_KEY`                 | `narration.open-ai.api-key`                 | `Your OpenAI API Key`                                                   |
| OpenAI Organization ID                                                                         | `NARRATION_OPEN_AI_ORGANIZATION_ID`         | `narration.open-ai.organization-id`         | `Your OpenAI Organization ID`                                           |
| Any additional properties to be passed to the Microsoft Semantic Kernel `OpenAIClientProvider` | `NARRATION_OPEN_AI_ADDITIONAL_PROPERTIES_*` | `narration.open-ai.additional-properties.*` | Any set of key/value pairs to be passed into the `OpenAIClientProvider` |

## Running Locally via Docker Compose
Pre-built images for this application can be found at [`quay.io/quarkus-super-heroes/rest-narration`](https://quay.io/repository/quarkus-super-heroes/rest-narration?tab=tags). 

Pick one of the versions of the application from the table below and execute the appropriate docker compose command from the `quarkus-super-heroes/rest-narration` directory.

| Description | Image Tag       | Docker Compose Run Command                                               |
|-------------|-----------------|--------------------------------------------------------------------------|
| JVM Java 17 | `java17-latest` | `docker compose -f deploy/docker-compose/java17.yml up --remove-orphans` |

[//]: # (| Native      | `native-latest` | `docker compose -f deploy/docker-compose/native.yml up --remove-orphans` |)

These Docker Compose files are meant for standing up this application only. If you want to stand up the entire system, [follow these instructions](../README.md#running-locally-via-docker-compose).

Once started the application will be exposed at `http://localhost:8087`.

## Deploying to Kubernetes
The application can be [deployed to Kubernetes using pre-built images](#using-pre-built-images) or by [deploying directly via the Quarkus Kubernetes Extension](#deploying-directly-via-kubernetes-extensions). Each of these is discussed below.

### Using pre-built images
Pre-built images for this application can be found at [`quay.io/quarkus-super-heroes/rest-narration`](https://quay.io/repository/quarkus-super-heroes/rest-narration?tab=tags).

Deployment descriptors for these images are provided in the [`deploy/k8s`](deploy/k8s) directory. There are versions for [OpenShift](https://www.openshift.com), [Minikube](https://quarkus.io/guides/deploying-to-kubernetes#deploying-to-minikube), [Kubernetes](https://www.kubernetes.io), and [Knative](https://knative.dev).

> [!NOTE]
> The [Knative](https://knative.dev/docs/) variant can be used on any Knative installation that runs on top of Kubernetes or OpenShift. For OpenShift, you need [OpenShift Serverless](https://docs.openshift.com/serverless/latest/about/about-serverless.html) installed from the OpenShift operator catalog. Using Knative has the benefit that services are scaled down to zero replicas when they are not used.

Pick one of the versions of the application from the table below and deploy the appropriate descriptor from the [`deploy/k8s` directory](deploy/k8s).

| Description | Image Tag       | OpenShift Descriptor                                      | Minikube Descriptor                                     | Kubernetes Descriptor                                       | Knative Descriptor                                    |
|-------------|-----------------|-----------------------------------------------------------|---------------------------------------------------------|-------------------------------------------------------------|-------------------------------------------------------|
| JVM Java 17 | `java17-latest` | [`java17-openshift.yml`](deploy/k8s/java17-openshift.yml) | [`java17-minikube.yml`](deploy/k8s/java17-minikube.yml) | [`java17-kubernetes.yml`](deploy/k8s/java17-kubernetes.yml) | [`java17-knative.yml`](deploy/k8s/java17-knative.yml) |

[//]: # (| Native      | `native-latest` | [`native-openshift.yml`]&#40;deploy/k8s/native-openshift.yml&#41; | [`native-minikube.yml`]&#40;deploy/k8s/native-minikube.yml&#41; | [`native-kubernetes.yml`]&#40;deploy/k8s/native-kubernetes.yml&#41; | [`native-knative.yml`]&#40;deploy/k8s/native-knative.yml&#41; |)

The application is exposed outside of the cluster on port `80`.

These are only the descriptors for this application only. If you want to deploy the entire system, [follow these instructions](../README.md#deploying-to-kubernetes).

### Deploying directly via Kubernetes Extensions
Following the [deployment section](https://quarkus.io/guides/deploying-to-kubernetes#deployment) of the [Quarkus Kubernetes Extension Guide](https://quarkus.io/guides/deploying-to-kubernetes) (or the [deployment section](https://quarkus.io/guides/deploying-to-openshift#build-and-deployment) of the [Quarkus OpenShift Extension Guide](https://quarkus.io/guides/deploying-to-openshift) if deploying to [OpenShift](https://openshift.com)), you can run one of the following commands to deploy the application and any of its dependencies (see [Kubernetes (and variants) resource generation](../docs/automation.md#kubernetes-and-variants-resource-generation) of the [automation strategy document](../docs/automation.md)) to your preferred Kubernetes distribution.

> [!NOTE]
> For non-OpenShift or minikube Kubernetes variants, you will most likely need to [push the image to a container registry](https://quarkus.io/guides/container-image#pushing) by adding the `-Dquarkus.container-image.push=true` flag, as well as setting the `quarkus.container-image.registry`, `quarkus.container-image.group`, and/or the `quarkus.container-image.name` properties to different values.

| Target Platform        | Java Version | Command                                                                                                                                                                                                                                      |
|------------------------|:------------:|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Kubernetes             |      17      | `./mvnw clean package -Dquarkus.profile=kubernetes -Dquarkus.kubernetes.deploy=true -DskipTests`                                                                                                                                             |
| OpenShift              |      17      | `./mvnw clean package -Dquarkus.profile=openshift -Dquarkus.container-image.registry=image-registry.openshift-image-registry.svc:5000 -Dquarkus.container-image.group=$(oc project -q) -Dquarkus.kubernetes.deploy=true -DskipTests`         |
| Minikube               |      17      | `./mvnw clean package -Dquarkus.profile=minikube -Dquarkus.kubernetes.deploy=true -DskipTests`                                                                                                                                               |
| Knative                |      17      | `./mvnw clean package -Dquarkus.profile=knative -Dquarkus.kubernetes.deploy=true -DskipTests`                                                                                                                                                |
| Knative (on OpenShift) |      17      | `./mvnw clean package -Dquarkus.profile=knative-openshift -Dquarkus.container-image.registry=image-registry.openshift-image-registry.svc:5000 -Dquarkus.container-image.group=$(oc project -q) -Dquarkus.kubernetes.deploy=true -DskipTests` |

You may need to adjust other configuration options as well (see [Quarkus Kubernetes Extension configuration options](https://quarkus.io/guides/deploying-to-kubernetes#configuration-options) and [Quarkus OpenShift Extension configuration options](https://quarkus.io/guides/deploying-to-openshift#configuration-reference)).

> [!TIP]
> The [`do_build` function in the `generate-k8s-resources.sh` script](../scripts/generate-k8s-resources.sh) uses these extensions to generate the manifests in the [`deploy/k8s` directory](deploy/k8s).
