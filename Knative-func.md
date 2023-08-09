# Native build with Knative functions

This doc will help you get started developing Spring Boot based functions using [Knative](https://knative.dev/docs/) [Functions](https://knative.dev/docs/functions/) project.
Knative Functions provides a simple programming model for using functions on Knative, without requiring in-depth knowledge of Knative, Kubernetes, containers, or dockerfiles.

Spring Boot based Knative functions can easily be built as GraalVM native images as we show below. 

# Installation

1. install Knative - https://knative.dev/docs/install/
1. install `kn` CLI - https://knative.dev/docs/client/install-kn/
1. install `func` CLI - https://knative.dev/docs/functions/install-func/

# The first function

## Crating the function

Run the following command to craete youe first `springboot` Knative `http` function:

```sh
func create kn-fun -l springboot -t http
cd kn-fun
```

## Building the function

Knative func supports multiple language packs and we are using the `springboot` language pack.
There are two types of function templates, `http` and `cloudevents`. We are staring off with the simpler `http` one.

Knative func uses Paketo buildpacks for building `springboot` functions. This works fine on Intel based systems but ARM based systems are currently not supported. Building your function with a buildpack is also a bit slow, so we prefer to build with [Jib](https://github.com/GoogleContainerTools/jib) to speed things up. Jib also works great on ARM based systems.

You can run the following to set the registry to use for the image and then build the function:

```sh
export FUNC_REGISTRY=docker.io/springdeveloper
./mvnw compile com.google.cloud.tools:jib-maven-plugin:3.3.2:build -Dimage=$FUNC_REGISTRY/kn-fun
```

> Note: The above command creates an `amd64` based image which you can deploy to an Intel based cluster. If you are on an ARM based system and want to use Kind as the deplyment cluster, then you can configure Jib to build multi-arch images. Add the following build plugin to the `pom.xml`.
```xml
        <plugin>
            <groupId>com.google.cloud.tools</groupId>
            <artifactId>jib-maven-plugin</artifactId>
            <version>3.3.2</version>
            <configuration>
            <from>
                <image>eclipse-temurin:17-jre</image>
                <platforms>
                <platform>
                    <architecture>amd64</architecture>
                    <os>linux</os>
                </platform>
                <platform>
                    <architecture>arm64</architecture>
                    <os>linux</os>
                </platform>
                </platforms>
            </from>
            </configuration>
        </plugin>
```

## Deploying the function

To deploy the function we use the `kn` CLI instead of the `func` CLI. The `func` CLI works best when you use it to both build and deploy the function. Since we already built and pushed the image it is easier and more flexible to use the `kn` CLI. Either way we end up with a Knative Service that runs our `springboot` function:

```sh
kn service create kn-fun --image $FUNC_REGISTRY/kn-fun
```

Once the function gets deployed we can access it with CURL. Make note of the URL that is displayed when deployment completes.

```sh
APP_URL=<the-url-for-deployment>
curl $APP_URL -H 'content-type: text/plain' -d SpringOne
```

## Building and deploying as a GraalVM native image

Once you are done developing your function, you most likely want to build it as a GraalVM native image since the startup time is so much quicker. The best way to do that with Knative functions is to push your code to a Git repository and build the image on the cluster.

### Prepare the cluster

There are two installation steps documented in the Knative Functions cluster build docs:

1. Install the Tekton Pipelines following the [Prerequisite](https://github.com/knative/func/blob/main/docs/building-functions/on_cluster_build.md#prerequisite) step.
1. Prepare the namespace you are using for on-cluster builds following the [Enabling a namespace to run Function related Tekton Pipelines](https://github.com/knative/func/blob/main/docs/building-functions/on_cluster_build.md#enabling-a-namespace-to-run-function-related-tekton-pipelines) step.

### Push your function code to a Git repo

You can create a Git repo with your function code using the following.

Create an empty public Git repository using GitHub UI or any other Git host and then set your Git remote.
As an example:

```sh
export GIT_REMOTE=git@github.com:trisberg/kn-fun.git
```

Then initalize your repo:

```sh
git init
git branch -M main
git remote add origin $GIT_REMOTE
```

Next, add your function and push to the Git repo:

```sh
git add .
git commit -m "initial commit"
git push --set-upstream origin main
```

# Build and Deploy as native image

Update the `func.yaml` file changing the following build env:

```yaml
build:
  buildEnvs:
  - name: BP_NATIVE_IMAGE
    value: "false"
```

to be `"true"`.

Set the GIT_URL as the https URL for your Git repo:

```sh
export GIT_URL=https://github.com/trisberg/kn-fun
```

Then deloy the function using:

```sh
func deploy -v --remote --registry $FUNC_REGISTRY --git-url $GIT_URL
```
