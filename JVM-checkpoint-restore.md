# Getting started with JVM Checkpoint/Restore builds

Spring reference docs has some coverage for [JVM Checkpoint Restore](https://docs.spring.io/spring-framework/reference/6.1/integration/checkpoint-restore.html). 

## Creating the project to use

We will use a snapshot version of Spring Boot and Spring Framework that has improved support for JVM Checkpoint/Restore.

We can use [start.spring.io](start.spring.io) to generate a new project to use:

```sh
wget 'https://start.spring.io/starter.zip?type=maven-project&language=java&bootVersion=3.2.0-SNAPSHOT&baseDir=demo&groupId=com.example&artifactId=demo&name=demo&description=Demo project for Spring Boot&packageName=com.example.demo&packaging=jar&javaVersion=17&dependencies=web,actuator' -O demo.zip
```

Extract the project and change to the `demo` directory:

```sh
unzip demo.zip
cd demo
```

## Add the `crac` dependency

We need to add an additional dependency to the `pom.xml` in the `<dependencies>` section:

```xml
		<dependency>
			<groupId>org.crac</groupId>
			<artifactId>crac</artifactId>
			<version>1.3.0</version>
		</dependency>
```

## Add a web-mvc controller

Create a Java source file for the controller:

```sh
touch src/main/java/com/example/demo/DemoController.java
```

Add the following content to the `src/main/java/com/example/demo/DemoController.java` file:

```
package com.example.demo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DemoController {

	private static Logger logger = LoggerFactory.getLogger(DemoController.class);

	@RequestMapping("/")
	public String index() {
		String ts = java.time.Clock.systemUTC().instant().toString();
		String message = "at " + ts;
		logger.info("Message: " + message);
		return "Hello " + message;
	}

}
```

## Add a Dockerfile

Since the CRaC feature relies on CRIU support in the Linux kernel, and we want to be able to run this on other systems, we do need to run this in a container.

> Note: The processor architecture needs to match the OpenJDK implementation, so we need to pass in an `ARCH` argument with the architecture. We need to specify `aarch64` for ARM based systems like Macs with M1/M2 chips or `x64` for AMD/Intel non-ARM systems.

Create a `Dockerfile` file in the root of the project with this content:

```
FROM ubuntu:22.04

# PROCESSOR_ARCH is either "aarch64" for ARM systems or "x64" for AMD/Intel systems
ARG ARCH

# Set ENV
ENV JAVA_HOME /azul-crac-jdk
ENV PATH $PATH:$JAVA_HOME/bin

# Add CRaC JDK
ADD https://cdn.azul.com/zulu/bin/zulu17.44.17-ca-crac-jdk17.0.8-linux_$ARCH.tar.gz $JAVA_HOME/openjdk.tar.gz
RUN tar --extract --file $JAVA_HOME/openjdk.tar.gz --directory "$JAVA_HOME" --strip-components 1; rm $JAVA_HOME/openjdk.tar.gz;

WORKDIR /home/app

# Add App
COPY target/demo-*.jar /home/app/demo.jar
COPY docker/entrypoint.sh /opt/app/entrypoint.sh

CMD ["/opt/app/entrypoint.sh"]
```

## Add the entrypoint script

We also need an `entrypoint.sh` script that can detect if we already have checkpoint files availabel. If that is the case we restore them, if not we need to start the app and then take a checkpoint.

Create a `docker` directory with an empty script file:

```sh
mkdir docker
touch docker/entrypoint.sh
chmod +x docker/entrypoint.sh
```

Next, edit the `docker/entrypoint.sh` file and add this content:

```
#!/bin/bash

CRAC_FILES_DIR=`eval echo ${CRAC_FILES_DIR}`
mkdir -p ${CRAC_FILES_DIR}

cd /home/app
if [ ! -f "$CRAC_FILES_DIR/READY" ]; then
  echo "Save checkpoint to $CRAC_FILES_DIR"
  ( echo 128 > /proc/sys/kernel/ns_last_pid ) 2>/dev/null || while [ $(cat /proc/sys/kernel/ns_last_pid) -lt 128 ]; do :; done
  java -Dmanagement.endpoint.health.probes.add-additional-paths="true" -Dmanagement.health.probes.enabled="true" -XX:CRaCCheckpointTo=$CRAC_FILES_DIR -jar /home/app/demo.jar &
  PID=$!
  sleep 5
  jcmd $PID JDK.checkpoint
  sleep 5
  echo "true" > $CRAC_FILES_DIR/READY
else
  echo "Restore checkpoint from $CRAC_FILES_DIR"
fi
exec java -Dmanagement.endpoint.health.probes.add-additional-paths="true" -Dmanagement.health.probes.enabled="true" -XX:CRaCRestoreFrom=$CRAC_FILES_DIR
```

## Build the docker image

We can now create the image by running:

```
case $(uname -m) in
    arm64)   arch="aarch64" ;;
    *)       arch="x64" ;;
esac
echo "Using CRaC enabled JDK with arch $arch"
./mvnw clean package -DskipTests --no-transfer-progress
docker build -t springdeveloper/demo:0.0.1 --build-arg ARCH=$arch .
```

## Run the docker image

When the image is built, we can run it locally. We need to define a volume to store the checkpoint files and we also ned to set some extra capabilities for checkpoint/restore to work. Run with the following command:

```sh
docker run -it --rm -p 8080:8080 -e CRAC_FILES_DIR=/crac/demo/0.0.1 --name demo \
  --mount source=cracvol,target=/crac \
  --cap-add CHECKPOINT_RESTORE --cap-add NET_ADMIN --cap-add SYS_PTRACE --cap-add SYS_ADMIN \
  springdeveloper/demo:0.0.1
```

You should see some log ouput like this:

```
Save checkpoint to /crac/demo/0.0.1

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::       (v3.2.0-SNAPSHOT)

2023-08-11T16:37:36.413Z  INFO 129 --- [           main] com.example.demo.DemoApplication         : Starting DemoApplication v0.0.1-SNAPSHOT using Java 17.0.8 with PID 129 (/home/app/demo.jar started by root in /home/app)
2023-08-11T16:37:36.414Z  INFO 129 --- [           main] com.example.demo.DemoApplication         : No active profile set, falling back to 1 default profile: "default"
2023-08-11T16:37:37.056Z  INFO 129 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8080 (http)
2023-08-11T16:37:37.062Z  INFO 129 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2023-08-11T16:37:37.062Z  INFO 129 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.11]
2023-08-11T16:37:37.125Z  INFO 129 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2023-08-11T16:37:37.126Z  INFO 129 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 682 ms
2023-08-11T16:37:37.424Z  INFO 129 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint(s) beneath base path '/actuator'
2023-08-11T16:37:37.456Z  INFO 129 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2023-08-11T16:37:37.469Z  INFO 129 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 1.249 seconds (process running for 1.768)
129:
CR: Checkpoint ...
/opt/app/entrypoint.sh: line 18:   129 Killed                  java -Dmanagement.endpoint.health.probes.add-additional-paths="true" -Dmanagement.health.probes.enabled="true" -XX:CRaCCheckpointTo=$CRAC_FILES_DIR -jar /home/app/demo.jar
2023-08-11T16:37:46.367Z  INFO 129 --- [Attach Listener] o.s.c.support.DefaultLifecycleProcessor  : Restarting Spring-managed lifecycle beans after JVM restore
2023-08-11T16:37:46.386Z  INFO 129 --- [Attach Listener] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2023-08-11T16:37:46.387Z  INFO 129 --- [Attach Listener] o.s.c.support.DefaultLifecycleProcessor  : Spring-managed lifecycle restart completed in 21 ms
```

We can see that after the app starts up, there is a checkpoint taken followed by the app getting killed and then starting up again restoring the state from the checkpoint files.

## Access the running app

From a different terminal run:

```sh
curl localhost:8080
```

```
Hello at 2023-08-11T16:17:54.991518Z
```

## Stop the docker container

We can just kill the running container:

```sh
docker kill demo  
```

## Run the docker image a second time

We use the same command for the second run:

```sh
docker run -it --rm -p 8080:8080 -e CRAC_FILES_DIR=/crac/demo/0.0.1 --name demo \
  --mount source=cracvol,target=/crac \
  --cap-add CHECKPOINT_RESTORE --cap-add NET_ADMIN --cap-add SYS_PTRACE --cap-add SYS_ADMIN \
  springdeveloper/demo:0.0.1
```

You should now see some log ouput like this:

```
Restore checkpoint from /crac/demo/0.0.1
2023-08-11T16:41:38.805Z  INFO 129 --- [Attach Listener] o.s.c.support.DefaultLifecycleProcessor  : Restarting Spring-managed lifecycle beans after JVM restore
2023-08-11T16:41:38.828Z  INFO 129 --- [Attach Listener] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2023-08-11T16:41:38.829Z  INFO 129 --- [Attach Listener] o.s.c.support.DefaultLifecycleProcessor  : Spring-managed lifecycle restart completed in 24 ms
```

That startup was much faster since we restored the state from the checkpoint files.

## Stop the docker container and remove the volume

When we are done, we can just kill the running container:

```sh
docker kill demo  
```

Finally, remove the volume:

```sh
docker volume rm cracvol
```
