![This tutorial is courtesy of Dash0](./images/dash0-logo.png)
# Monitoring Heroku Applications with OpenTelemetry

Heroku makes it easy to deploy and run applications, handling much of the operational overhead. One of the time-consuming aspects of running a system is setting up how to monitor it. That’s why, at Dash0, we're excited to hear that Heroku's next-generation Fir platform natively integrates OpenTelemetry (OTel) for observability. The current Cedar platform does not have these facilities, but it is nevertheless possible to monitor your apps running on it with OpenTelemetry.

This repository provides a demo application to guide you through monitoring a Heroku Cedar app with OpenTelemetry and Cloud Native Buildpacks. For a comprehensive walkthrough with detailed explanations see the full guide on our website; however, all the core steps required to deploy the application and start monitoring are also outlined below.

## What is Heroku's Cedar platform?
Heroku's Cedar platform provides the runtime environment for your apps by abstracting the operating system, web server, and process management. While Cedar doesn’t natively support Cloud Native Buildpacks, you can still deploy production-grade container images by building them yourself and pushing to Heroku’s Container Registry.

## What is Spring Boot?
Spring Boot is a framework for building stand‑alone, production‑grade Spring applications with minimal configuration. It packages your app as an executable JAR with an embedded servlet container (Tomcat, Jetty, or Undertow) and provides out‑of‑the‑box features like health checks, metrics, and externalized configuration.

## What are Cloud Native Buildpacks?
Writing Dockerfiles is easy; writing and maintaining production‑grade Dockerfiles is hard. Cloud Native Buildpacks (CNBs) like Paketo offer composable building blocks to create secure, performant container images without hand‑crafting Dockerfiles. With CNBs, running a single command (e.g. `./mvnw spring-boot:build-image`) can yield a best‑practice image including a tuned JVM, SSL certs, and observability agents.

## What is OpenTelemetry (OTel)?
OpenTelemetry is the CNCF observability project that standardizes collection of traces, metrics, and logs. By adding the OpenTelemetry Java agent as a buildpack layer, you can auto‑instrument your Java app with no code changes and export telemetry to any OTLP‑compatible backend.

## Introducing the Demo Application
We'll use a simple Spring Boot app in this repo. Our goal is to use Paketo buildpacks to inject the OpenTelemetry Java agent into the image, then deploy that image to Heroku’s Container Registry.

## Prerequisites
- A Heroku account and the Heroku CLI
- Docker installed locally
- Java 21+ and Maven (or use the Maven Wrapper)

## 1. Create a Heroku App with the Container Stack
```shell
heroku create <YOUR_APP_NAME> --stack container
# For existing apps:
# heroku stack:set container -a <YOUR_APP_NAME>
```

## 2. Configure Buildpacks in `pom.xml`
Ensure your `pom.xml` includes the Spring Boot plugin configuration:

```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <configuration>
    <image>
      <name>registry.heroku.com/${HEROKU_APP_NAME}/web</name>
      <builder>paketobuildpacks/builder-jammy-base</builder>
      <imagePlatform>linux/amd64</imagePlatform>  
      <buildpacks>
        <buildpack>docker.io/paketobuildpacks/java</buildpack>
        <buildpack>docker.io/paketobuildpacks/opentelemetry:2.12.0</buildpack>
      </buildpacks>
      <env>
        <BP_JVM_VERSION>21</BP_JVM_VERSION>
        <BP_OPENTELEMETRY_ENABLED>true</BP_OPENTELEMETRY_ENABLED>
        <BP_INCLUDE_FILES>src/main/resources/sdk-config.yaml</BP_INCLUDE_FILES>
        <BPE_DEFAULT_OTEL_JAVAAGENT_ENABLED>true</BPE_DEFAULT_OTEL_JAVAAGENT_ENABLED>
        <BPE_OVERRIDE_OTEL_EXPERIMENTAL_CONFIG_FILE>/workspace/BOOT-INF/classes/sdk-config.yaml</BPE_OVERRIDE_OTEL_EXPERIMENTAL_CONFIG_FILE>
      </env>
    </image>
  </configuration>
</plugin>
```

## 3. Prepare the OpenTelemetry SDK Config File
The `sdk-config.yaml` in `src/main/resources` defines resource attributes for Heroku Dyno metadata:

```yaml
# Configure resource attributes for all signals.
resource:
  attributes:
    - name: cloud.provider
      value: heroku
    - name: heroku.app.id
      value: ${HEROKU_APP_ID:-unknown-app-id}
    #- name: heroku.release.commit
    #  value: ${HEROKU_BUILD_COMMIT:-unknown-release-commit}
    #  (HEROKU_SLUG_COMMIT is deprecated per Heroku Dyno Metadata docs)
    - name: heroku.release.creation_timestamp
      value: ${HEROKU_RELEASE_CREATED_AT:-unknown-release-created-at}
    - name: service.name
      value: ${HEROKU_APP_NAME:-heroku-demo}
    - name: service.version
      value: ${HEROKU_RELEASE_VERSION:-v0}
    - name: service.instance.id
      value: ${HEROKU_DYNO_ID:-unknown-instance-id}
```

Enable the Dyno Metadata labs before deploying:

```shell
heroku labs:enable runtime-dyno-metadata -a <YOUR_APP_NAME>
heroku labs:enable runtime-dyno-build-metadata -a <YOUR_APP_NAME>
```

Please note that it may take a few moments for these changes to take effect after the commands have been executed.

## 4. Set Required Config Vars
Configure JVM settings and OpenTelemetry endpoint/headers:

```shell
heroku config:set \
  OTEL_EXPORTER_OTLP_ENDPOINT=https://ingress.<REGION>.aws.dash0.com \
  OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <YOUR_DASH0_API_TOKEN>" \
  -a <YOUR_APP_NAME>
```

For Dash0, you will find the values to set for the `OTEL_EXPORTER_OTLP_ENDPOINT` and `OTEL_EXPORTER_OTLP_HEADERS` in the [OpenTelemetry Java onboarding instructions](https://app.dash0.com/onboarding/instructions/programming-languages/java).

## 5. Build the Container Image
Use the Spring Boot Maven plugin to build the application container image (including the OpenTelemetry Java agent):

```shell
./mvnw package spring-boot:build-image
```

## 6. Push and Release to Heroku
Log in to the Container Registry, push, and release:

```shell
heroku container:login
# Push and release
docker push registry.heroku.com/<YOUR_APP_NAME>/web
heroku container:release web -a <YOUR_APP_NAME>
```

Once the new container image has been released and Heroku has deployed it, you should see logs, traces, and metrics appear in your configured observability solution when you open the application.

## 7. Ready to Explore in Dash0

Since you have gotten it this far, how about trying this demo with [Dash0](https://www.dash0.com/), the OpenTelemetry-native tool that makes observability simple and enjoyable?
There's a free trial waiting for you, 14 days, no questions asked: https://www.dash0.com/sign-up
