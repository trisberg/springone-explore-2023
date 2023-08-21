# Tanzu Academy - Developer Sandbox Challenge

This challenge will get you familiar with the steps needed to run a native build on Tanzu Application Platform (TAP). We use the basics from the [Native build with TAP](https://github.com/trisberg/springone-explore-2023/blob/main/TAP-native-build.md) demo, but here we use a different accelerator, the `tanzu-java-web-app` one. If you know how to "copy-and-paste", then you should be all set for this challenge.

This is a "cheat sheet" for the commands you need to run:

1. Sign up and start a new sandbox lab
    - https://tanzu.academy/guides/developer-sandbox
2. Copy the link to "Tanzu Developer Portal" provided under "Getting started".
3. Set the env var `ACC_SERVER_URL` to the link you just copied (replace the URL in the example with the one you just copied)
    ```sh
    export ACC_SERVER_URL=https://tap-gui.tapb-engaging-sunbird.tapsandbox.com/
    ```
4. Generate a project from the spring-cloud-serverless accelerator
    ```sh
    tanzu accelerator generate tanzu-java-web-app --options '{"buildTool" : "maven",  "javaVersion" : "17",  "nativeBuild" : true,  "projectName" : "hello-native",  "springBootVersion" : "3.1"}'
    ```
5. Unzip and cd to the app directory
    ```sh
    unzip hello-native.zip
    cd hello-native
    ```
6. Deploy the native workload
    ```sh
    tanzu apps workload create hello-native -y -f config/workload-native.yaml --local-path .
    ```
7. Play some elevator music while you wait for the build to finish (~15 min)
    ```sh
    tanzu apps workload get hello-native
    ```
8. When the app becomes ready it will show a URL, capture that URL
    ```sh
    APP_URL=$(kubectl get service.serving.knative.dev/hello-native -ojsonpath='{.status.url}')
    ```
9. Invoke the app using `curl`
    ```sh
    curl -w'\n' $APP_URL
    ```

