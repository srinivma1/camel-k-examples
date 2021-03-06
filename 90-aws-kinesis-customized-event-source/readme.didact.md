# Camel Knative Source Basic Example

This example demonstrates how to get started with Camel based Knative sources by showing you some of the most important
features that you should know before trying to develop more complex examples. This extends the ideas presented on the [AWS Kinesis Source Basic](../04-aws-kinesis-source-basic) example and shows how to customize the Kinesis client. In this example we
show how to configure a custom Kinesis client for testing using a [LocalStack](https://github.com/localstack/localstack) instance
instead of a real AWS Kinesis instance. Although this example focuses on a general scenario aimed at testing, the overall idea and design is also applicable for scenarios when a custom client configuration is necessary.

You can find more information about Apache Camel and Apache Camel K on the [official Camel website](https://camel.apache.org).

## Before you begin

Read the general instructions in the [root README.md file](../README.md) for setting up your environment and the Kubernetes cluster before looking at this example.

Make sure you've read the [installation instructions](https://camel.apache.org/camel-k/latest/installation/installation.html) for your specific
cluster before starting the example.

You should open this file with [Didact](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-didact) if available on your IDE.

You need a Maven repository where the custom client can be deployed.

## Requirements

<a href='didact://?commandId=vscode.didact.validateAllRequirements' title='Validate all requirements!'><button>Validate all Requirements at Once!</button></a>

**Kubectl CLI**

The Kubernetes `kubectl` CLI tool will be used to interact with the Kubernetes cluster.

[Check if the Kubectl CLI is installed](didact://?commandId=vscode.didact.cliCommandSuccessful&text=kubectl-requirements-status$$kubectl%20help&completion=Checked%20kubectl%20tool%20availability "Tests to see if `kubectl help` returns a 0 return code"){.didact}

*Status: unknown*{#kubectl-requirements-status}

**Connection to a Kubernetes cluster**

You need to connect to a Kubernetes cluster in order to run the example.

[Check if you're connected to a Kubernetes cluster](didact://?commandId=vscode.didact.cliCommandSuccessful&text=cluster-requirements-status$$kubectl%20get%20pod&completion=Checked%20Kubernetes%20connection "Tests to see if `kubectl get pod` returns a 0 return code"){.didact}

*Status: unknown*{#cluster-requirements-status}

**Apache Camel K CLI ("kamel")**

You need the Apache Camel K CLI ("kamel") in order to access all Camel K features.

[Check if the Apache Camel K CLI ("kamel") is installed](didact://?commandId=vscode.didact.requirementCheck&text=kamel-requirements-status$$kamel%20version$$Camel%20K%20Client&completion=Checked%20if%20Camel%20K%20CLI%20is%20available%20on%20this%20system. "Tests to see if `kamel version` returns a result"){.didact}

*Status: unknown*{#kamel-requirements-status}

**Knative installed on the cluster**

The cluster also needs to have Knative installed and working. Refer to the [official Knative documentation](https://knative.dev/docs/install/) for information on how to install it in your cluster.

[Check if the Knative Serving is installed](didact://?commandId=vscode.didact.requirementCheck&text=kserving-project-check$$kubectl%20api-resources%20--api-group=serving.knative.dev$$kservice%2Cksvc&completion=Verified%20Knative%20services%20installation. "Verifies if Knative Serving is installed"){.didact}

*Status: unknown*{#kserving-project-check}

[Check if the Knative Eventing is installed](didact://?commandId=vscode.didact.requirementCheck&text=keventing-project-check$$kubectl%20api-resources%20--api-group=messaging.knative.dev$$inmemorychannels&completion=Verified%20Knative%20eventing%20services%20installation. "Verifies if Knative Eventing is installed"){.didact}

*Status: unknown*{#keventing-project-check}

**Knative Camel Source installed on the cluster**

The cluster also needs to have installed the Knative Camel Source from the camel.yaml in the [Eventing Sources release page](https://github.com/knative/eventing-contrib/releases)

[Check if the Knative Camel Source is installed](didact://?commandId=vscode.didact.requirementCheck&text=kservice-project-check$$kubectl%20api-resources%20--api-group=sources.knative.dev$$camelsources&completion=Verified%20Knative%20Camel%20Source%20installation. "Verifies if Knative Camel Source is installed"){.didact}

*Status: unknown*{#kservice-project-check}

### Optional Requirements

The following requirements are optional. They don't prevent the execution of the demo, but may make it easier to follow.

**VS Code Extension Pack for Apache Camel**

The VS Code Extension Pack for Apache Camel provides a collection of useful tools for Apache Camel K developers,
such as code completion and integrated lifecycle management. They are **recommended** for the tutorial, but they are **not**
required.

You can install it from the VS Code Extensions marketplace.

[Check if the VS Code Extension Pack for Apache Camel by Red Hat is installed](didact://?commandId=vscode.didact.extensionRequirementCheck&text=extension-requirement-status$$redhat.apache-camel-extension-pack&completion=Camel%20extension%20pack%20is%20available%20on%20this%20system. "Checks the VS Code workspace to make sure the extension pack is installed"){.didact}

*Status: unknown*{#extension-requirement-status}

## 1. First steps

Let's open a terminal and go to the example directory:

```
cd 90-aws-kinesis-customized-event-source
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$cd%2090-aws-kinesis-customized-event-source&completion=Executed%20command. "Opens a new terminal and sends the command above"){.didact})

We're going to create a namespace named `aws-kinesis-customized-event-source` for running the example. To create it, execute the following command:

```
kubectl create namespace aws-kinesis-customized-event-source
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kubectl%20create%20namespace%20aws-kinesis-customized-event-source&completion=New%20project%20creation. "Opens a new terminal and sends the command above"){.didact})

Now we can set the `aws-kinesis-customized-event-source` namespace as default namespace for the following commands:

```
kubectl config set-context --current --namespace=aws-kinesis-customized-event-source
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kubectl%20config%20set-context%20--current%20--namespace%3Daws-kinesis-customized-event-source&completion=New%20project%20creation. "Opens a new terminal and sends the command above"){.didact})

You need to install Camel K in the `aws-kinesis-customized-event-source` namespace (or globally in the whole cluster).
For this example to run, you must configure Camel K to include your Maven repository in its configuration, so that
dependencies can be downloaded from it. To do so, execute the following command replacing `https://url.of.the.repository`
with the actual URL of the repository:

```
kamel install --maven-repository https://url.of.the.repository
```

NOTE: The `kamel install` command requires some prerequisites to be successful in some situations, e.g. you need to enable the registry addon on Minikube. Refer to the [Camel K install guide](https://camel.apache.org/camel-k/latest/installation/installation.html) for cluster-specific instructions.

To check that Camel K is installed we'll retrieve the IntegrationPlatform object from the namespace:

```
kubectl get integrationplatform
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kubectl%20get%20integrationplatform&completion=Executed%20Command. "Opens a new terminal and sends the command above"){.didact})

You should find an IntegrationPlatform in status `Ready`.

You can now proceed to the next section.


## 2. Deploy the custom client to a Maven repository


Deploy the custom client to a maven repository, adjusting the command below so that the
repository ID and the URL matches the ones used in your Maven repository.

```
mvn -f extra/custom-kinesis-configuration/pom.xml -DaltDeploymentRepository="id-of-the-repository::default::https://url.of.the.repository" deploy
```

## 3. Deploy LocalStack in the cluster

Deploy the a LocalStack instance for testing:

```
kubectl apply -f extra/localstack.yaml
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kubectl%20apply%20-f%20extra%2Flocalstack.yaml&completion=deployment%20%22localstack%22%20created. "Create a LocalStack deployment"){.didact})


## 4. Preparing the environment

This repository contains a simple [aws-kinesis.properties](didact://?commandId=vscode.openFolder&projectFilePath=05-aws-kinesis-source-advanced/aws-kinesis.properties&completion=Opened%20the%aws-kinesis.properties%20file "Opens the aws-kinesis.properties file"){.didact} that contains the access key and secret key for accessing the AWS Kinesis stream.

```
kubectl create secret generic aws-kinesis --from-file=aws-kinesis.properties
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kubectl%20create%20secret%20generic%20aws-kinesis%20--from-file%3Daws-kinesis.properties&completion=secret%20%22aws-kinesis%22%20created. "Create a secret with AWS Kinesis credentials"){.didact})

As the example levareges [Knative Eventing channels](https://knative.dev/docs/eventing/channels/), we need to create the one that the example will use:

```
kubectl apply -f aws-kinesis-channel.yaml
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kubectl%20apply%20-f%20aws-kinesis-channel.yaml&completion=inmemorychannel.messaging.knative.dev/aws-kinesis$20created. "Create a Knative InMemoryChannel named aws-kinesis"){.didact})


## 5. Running a Camel Source

This repository contains a simple Camel Source based on the [AWS Kinesis component](https://camel.apache.org/components/latest/aws-kinesis-component.html) that forward streaming events received on the AWS Kinesis stream to a Knative channel named `aws-kinesis`.

Use the following command to deploy the Camel Source:

```
kubectl apply -f aws-kinesis-source.yaml
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kubectl%20apply%20-f%20aws-kinesis-source.yaml&completion=camelsource.sources.knative.dev/camel-aws-kinesis-source%20created. "Opens a new terminal and sends the command above"){.didact})

## 6. Running a basic integration to create Kinesis events for consumption by the Camel Source

You need a producer adding data to a Kinesis stream to try this example. This integration
comes with a sample producer that will send 100 messages with the text `Hello Camel K`
every 3 seconds.

```
kamel run --secret aws-kinesis --dependency mvn:org.apache.camel.k.examples:custom-kinesis-configuration:1.0.3 --property camel.component.aws-kinesis.configuration=#class:org.apache.camel.k.examples.CustomKinesisConfiguration  --property amazon.host=localstack:4568 aws-kinesis-producer.groovy
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kamel%20run%20--secret%20aws-kinesis%20--dependency%20mvn:org.apache.camel.k.examples:custom-kinesis-configuration:1.0.3%20--property%20camel.component.aws-kinesis.configuration=%23class:org.apache.camel.k.examples.CustomKinesisConfiguration%20--property%20amazon.host=localstack:4568%20aws-kinesis-producer.groovy&completion=Ran%20the%20AWS%20Kinesis%20producer. "Opens a terminal and runs the command above"){.didact})

If everything is ok, after the build phase finishes, you should see the Camel integration running.

## 7. Running a basic integration to forward Kinesis events to the console

```
kamel run aws-kinesis-consumer.groovy --dev
```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kamel%20run%20aws-kinesis-consumer.groovy%20--dev&completion=Camel%20K%20aws-kinesis-consumer%20integration%20run%20in%20dev%20mode. "Opens a new terminal and sends the command above"){.didact})

If everything is ok, after the build phase finishes, you should see the Camel integration running.


## 8. Uninstall

To cleanup everything, execute the following command:

```kubectl delete namespace aws-kinesis-customized-event-source```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kubectl%20delete%20namespace%20aws-kinesis-customized-event-source&completion=Removed%20the%20namespace%20from%20the%20cluster. "Cleans up the cluster after running the example"){.didact})