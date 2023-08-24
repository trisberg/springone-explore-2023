# springone-explore-2023

SpringOne at VMware Explore 2023 :: Scaling Your Spring Boot App to Zero

## Link to session

[https://springone.io/sessions/scaling-your-spring-boot-app-to-zero](https://springone.io/sessions/scaling-your-spring-boot-app-to-zero)

## Slides

[SpringOne at VMware Explore 2023 Scaling Your Spring Boot App to Zero](SpringOne_at_VMware_Explore_2023_Scaling_Your_Spring_Boot_App_to_Zero.pdf)


## Tools to install

1. [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
1. [Tanzu CLI](https://github.com/vmware-tanzu/tanzu-cli/blob/main/docs/quickstart/install.md#from-the-binary-releases-in-github-project)
1. [TAP CLI plugins](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.6/tap/install-tanzu-cli.html#install-tanzu-cli-plugins-5)
1. [Carvel](https://carvel.dev/)
1. [VS Code](https://code.visualstudio.com/download)
1. [VS Code - Tanzu Developer Tools](https://marketplace.visualstudio.com/items?itemName=vmware.tanzu-dev-tools)
1. [VS Code - Tanzu App Accelerator](https://marketplace.visualstudio.com/items?itemName=vmware.tanzu-app-accelerator)

## Demos

### GraalVM Native

1. [Native build with Knative functions](Knative-func.md)
1. [Native build with TAP](TAP-native-build.md)

### JVM Checkpoint/Restore

1. [Getting started with JVM Checkpoint/Restore builds](JVM-checkpoint-restore.md)
1. [Using JVM Checkpoint/Restore on Knative](Knative-checkpoint-restore.md)
1. [Using JVM Checkpoint/Restore on TAP](TAP-checkpoint-restore.md)

## Challenge

1. [Tanzu Academy - Developer Sandbox Challenge](Sandbox-challenge.md)

## Links

### JVM Checkpoint/Restore demo app

- https://github.com/trisberg/hello-world

This app has instructions for running it:
- locally using Docker
- kubernetes deployment/service
    - including [notes](https://github.com/trisberg/hello-world/blob/main/keda/README.md) for scaling with Keda
- knative service

### Tanzu Academy - Developer Sandbox

- https://bit.ly/TanzuDevTry
- https://tanzu.academy/guides/developer-sandbox

<img src="images/bit.ly_TanzuDevTry.png" alt="QR Code" width="300"/>

### This repo

- https://github.com/trisberg/springone-explore-2023

![QR Code](images/springone-explore-2023.png)
