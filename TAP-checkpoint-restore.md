# Using JVM Checkpoint/Restore on TAP

## Prepare your TAP cluster

There are some modifications needed for the cluster, see the [Modifying TAP cluster when using JVM Checkpoint/Restore](TAP-checkpoint-restore-modifications.md) document for details.

## Create a workload file

We are using the `hello-world` app from https://github.com/trisberg/hello-world.git Git repo. You can clone that repo and build your own image or use a pre-built `springdeveloper/hello-world:amd64-0.1.0` image.

```sh
mkdir hello-world
cd hello-world
mkdir -p config
cat << EOF > config/workload.yaml
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  name: hello-world
  labels:
    apps.tanzu.vmware.com/workload-type: web
    app.kubernetes.io/part-of: hello-world
spec:
  env:
  - name: APP_MESSAGE
    value: World
  - name: APP_VERSION
    value: 0.1.0
  - name: NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
  - name: CRAC_FILES_DIR
    value: /var/crac/\${APP_VERSION}/\${NODE_NAME}
  params:
  - name: annotations
    value:
      autoscaling.knative.dev/minScale: "0"
  image: springdeveloper/hello-world:amd64-0.1.0
EOF
```

## Deploy the workload

This will craete the workload base on the content in `config/workload.yaml`.

Run:

```sh
tanzu apps workload apply -f config/workload.yaml
```

Now you can get the status from the deployment using:

```sh
tanzu apps workload get hello-world
```

This could take a minute or two, but you should soon see the app deployed.

## Access the app

Once the Knative service is available you should be able to capture the URL by running:

```sh
APP_URL=$(kubectl get service.serving.knative.dev/hello-world -ojsonpath='{.status.url}')
```

Then you can use CURL to invoke the app:

```sh
curl $APP_URL
```

You should see a message like the following:

```sh
% curl $APP_URL                                                                            
Hello World from hello-world-00001-deployment-77df7db96f-wcnlm at 2023-08-09T21:02:49.716763576Z
```

## Check the startup logs of the app

You can access the logs by running:

```sh
kubectl logs -l=app.kubernetes.io/component=run,app.kubernetes.io/part-of=hello-world -c workload
```

The first time the app is started on a node you will see logs like the following. The logs show the first start followed by a checkpoint and a restart of the app, now restoring from the checkpoit files.

```
Save checkpoint to /var/crac/0.1.0/gke-tap-fit-lion-default-pool-3138552f-f40x

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::       (v3.2.0-SNAPSHOT)

2023-08-09T21:05:30.600Z  INFO 129 --- [           main] c.example.hello.HelloWorldApplication    : Starting HelloWorldApplication v0.1.0 using Java 17.0.8 with PID 129 (/home/app/BOOT-INF/classes started by root in /home/app)
2023-08-09T21:05:30.613Z  INFO 129 --- [           main] c.example.hello.HelloWorldApplication    : No active profile set, falling back to 1 default profile: "default"
2023-08-09T21:05:33.724Z  INFO 129 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8080 (http)
2023-08-09T21:05:33.747Z  INFO 129 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2023-08-09T21:05:33.748Z  INFO 129 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.11]
2023-08-09T21:05:33.904Z  INFO 129 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2023-08-09T21:05:33.908Z  INFO 129 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 3064 ms
2023-08-09T21:05:34.343Z  INFO 129 --- [           main] c.example.hello.HelloWorldConfiguration  : app.message: World
2023-08-09T21:05:34.345Z  INFO 129 --- [           main] c.example.hello.HelloWorldConfiguration  : CRAC_FILES_DIR: /var/crac/0.1.0/gke-tap-fit-lion-default-pool-3138552f-f40x
2023-08-09T21:05:35.303Z  INFO 129 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint(s) beneath base path '/actuator'
2023-08-09T21:05:35.507Z  INFO 129 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2023-08-09T21:05:35.575Z  INFO 129 --- [           main] c.example.hello.HelloWorldApplication    : Started HelloWorldApplication in 5.818 seconds (process running for 7.929)
2023-08-09T21:05:35.799Z  INFO 129 --- [nio-8080-exec-5] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2023-08-09T21:05:35.801Z  INFO 129 --- [nio-8080-exec-5] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2023-08-09T21:05:35.806Z  INFO 129 --- [nio-8080-exec-5] o.s.web.servlet.DispatcherServlet        : Completed initialization in 3 ms
2023-08-09T21:05:35.885Z  INFO 129 --- [nio-8080-exec-5] com.example.hello.HelloWorldController   : Message: World from hello-world-00001-deployment-77df7db96f-b6gxm at 2023-08-09T21:05:35.884715458Z
129:
CR: Checkpoint ...
/opt/app/entrypoint.sh: line 17:   129 Killed                  java -Dmanagement.endpoint.health.probes.add-additional-paths="true" -Dmanagement.health.probes.enabled="true" -XX:CRaCCheckpointTo=$CRAC_FILES_DIR org.springframework.boot.loader.JarLauncher
2023-08-09T21:05:44.510Z  INFO 129 --- [Attach Listener] o.s.c.support.DefaultLifecycleProcessor  : Restarting Spring-managed lifecycle beans after JVM restore
2023-08-09T21:05:44.559Z  INFO 129 --- [Attach Listener] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2023-08-09T21:05:44.563Z  INFO 129 --- [Attach Listener] o.s.c.support.DefaultLifecycleProcessor  : Spring-managed lifecycle restart completed in 53 ms
```

Any subsequent starts will just show the restoring from the checkpoint files and startup is :

```
2023-08-09T21:11:57.069Z  INFO 129 --- [           main] c.example.hello.HelloWorldConfiguration  : app.message: World
2023-08-09T21:11:57.071Z  INFO 129 --- [           main] c.example.hello.HelloWorldConfiguration  : CRAC_FILES_DIR: /var/crac/0.1.0/gke-tap-fit-lion-default-pool-3138552f-f40x
2023-08-09T21:11:58.029Z  INFO 129 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint(s) beneath base path '/actuator'
2023-08-09T21:11:58.307Z  INFO 129 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2023-08-09T21:11:58.334Z  INFO 129 --- [           main] c.example.hello.HelloWorldApplication    : Started HelloWorldApplication in 5.539 seconds (process running for 7.286)
2023-08-09T21:11:58.489Z  INFO 129 --- [nio-8080-exec-2] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2023-08-09T21:11:58.502Z  INFO 129 --- [nio-8080-exec-2] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2023-08-09T21:11:58.505Z  INFO 129 --- [nio-8080-exec-2] o.s.web.servlet.DispatcherServlet        : Completed initialization in 2 ms
2023-08-09T21:11:58.572Z  INFO 129 --- [nio-8080-exec-2] com.example.hello.HelloWorldController   : Message: World from hello-world-00001-deployment-77df7db96f-lkbtx at 2023-08-09T21:11:58.571799596Z
```
