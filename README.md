# VProfile Application

A Java web application packaged as a WAR and organized around a Spring MVC/Spring Security application layer, JSP views, and integrations configured for MySQL, Memcached, RabbitMQ, and Elasticsearch. The repository also includes container build definitions and a GitHub Actions workflow for pull-request checks and an AWS ECR/Helm GitOps update on pushes to `main`.

> **Configuration and security note:** The checked-in `application.properties` and database Dockerfile contain development credentials and fixed service hostnames. Treat these values as examples only; replace them and move secrets out of source control before using this project in any shared or production environment.

## Table of contents

- [Project overview and architecture](#project-overview-and-architecture)
- [Technology stack](#technology-stack)
- [Repository structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Build and test with Maven](#build-and-test-with-maven)
- [Docker images](#docker-images)
- [Configuration and environment](#configuration-and-environment)
- [CI/CD workflow](#cicd-workflow)
- [SonarQube integration](#sonarqube-integration)
- [Deployment workflow](#deployment-workflow)
- [Troubleshooting](#troubleshooting)

## Project overview and architecture

The project is a traditional Java web application, not a Spring Boot executable JAR. Maven packages it as a WAR (`com.visualpathit:vprofile:v2`), with Java sources in `src/main/java` and a JSP-based web application under `src/main/webapp`. Spring XML configuration files in `WEB-INF` wire MVC, data access, RabbitMQ, and security configuration; `application.properties` is loaded from the classpath.

At a high level:

```text
Browser
  │ HTTP
  ▼
Nginx reverse proxy (optional container; listens on port 80)
  │ proxies to vproapp:8080
  ▼
Tomcat 10 hosting the VProfile WAR (application container; port 8080)
  ├── MySQL: accounts database (configured host: vprodb:3306)
  ├── Memcached (configured active host: vprocache01:11211)
  ├── RabbitMQ (configured host: vpromq01:5672)
  └── Elasticsearch (configured host: localhost:9300)
```

The repository contains Dockerfiles for the app, database, and Nginx web tier, but does **not** contain a Docker Compose file or definitions for the Memcached, RabbitMQ, or Elasticsearch services. The diagram reflects configuration and container files in this repository; it is not a complete, self-contained deployment topology.

The application source includes account/user and role models, repositories, services, controllers, validation, login/registration and user JSP views, file upload, and controller tests. Controllers also exist for RabbitMQ and Elasticsearch-related operations.

## Technology stack

| Area | Verified technologies/configuration |
|---|---|
| Language and build | Java (Maven compiler source/target 17), Maven, WAR packaging |
| Web layer | Spring Framework 6.0.11, Spring MVC, JSP, Jakarta Servlet APIs |
| Security and persistence | Spring Security 6.1.2, Spring Data JPA 3.1.2, Hibernate ORM (declared 7.0.0.Alpha3), MySQL Connector/J 8.0.33 |
| Messaging and supporting services | Spring Rabbit / RabbitMQ client, Elasticsearch Java client 7.10.2, spymemcached |
| Web/runtime containers | Tomcat 10 with JDK 21 in app Dockerfiles; Nginx; MySQL 8.0.33 in database Dockerfile |
| Tests and quality | JUnit 4 and JUnit 5 dependencies, Mockito, Checkstyle invocation and SonarQube scan in GitHub Actions |
| CI/CD | GitHub Actions, AWS ECR, Helm values update in a separate repository |

Dependency and tool versions above are the versions declared in the repository files; this table does not imply compatibility or support guarantees.

## Repository structure

```text
.
├── .github/workflows/ci.yml          # PR checks and push-triggered image/GitOps workflow
├── Docker-files/
│   ├── app/Dockerfile                # Builds image from target/vprofile-v2.war
│   ├── app/multistage/Dockerfile     # Multi-stage Docker build definition
│   ├── db/Dockerfile                 # MySQL image and initialization SQL
│   └── web/                          # Nginx image and reverse-proxy configuration
├── pom.xml                           # Maven dependencies, plugins, and WAR build
├── sonar-project.properties          # SonarQube project and report paths
└── src/
    ├── main/java/com/visualpathit/   # Java account/application source
    ├── main/resources/               # application.properties, SQL, logging, validation
    ├── main/webapp/WEB-INF/          # Spring XML, web.xml, and JSP views
    └── test/java/                    # Controller and model tests
```

## Prerequisites

- Git
- JDK 17 for the project’s declared compiler source/target; use JDK 21 if you want to match the workflow's configured JDK and Tomcat container runtime
- Maven 3.x (or the Maven Wrapper if one is added; no wrapper is present in this repository)
- Docker Engine for container builds and runs
- Access to the dependent services, with endpoints matching the application configuration or adjusted for your environment

The GitHub Actions workflow explicitly installs Temurin JDK 21. Local Maven compiles with source and target 17 as declared in `pom.xml`.

## Build and test with Maven

From the repository root:

```bash
mvn clean verify
```

This cleans, compiles, runs the configured test lifecycle, and packages the WAR. The expected artifact name based on the Maven coordinates and the checked-in Dockerfile is:

```text
target/vprofile-v2.war
```

To run the same Checkstyle report goal used in the pull-request workflow:

```bash
mvn clean verify checkstyle:checkstyle -B
```

The workflow calls `checkstyle:checkstyle` to generate a report; the command shown does not by itself establish a Checkstyle quality gate. To run locally with the configured Jetty Maven plugin, use:

```bash
mvn jetty:run
```

The plugin sets the web context path to `/`. Application startup and useful functionality still depend on reachable services and valid configuration. The repository does not document a ready-made local service stack.

## Docker images

### Application image from a local Maven build

Build the WAR first, then build the application image using the Dockerfile that copies that artifact:

```bash
mvn clean package

docker build \
  -f Docker-files/app/Dockerfile \
  -t vprofile-app:local \
  .
```

Run the container with the Tomcat port published:

```bash
docker run --rm --name vprofile-app \
  -p 8080:8080 \
  vprofile-app:local
```

Open `http://localhost:8080/`. The container alone does not provide the configured database, cache, message broker, or search service. Ensure those endpoints are available before expecting application features that use them to work.

### Multi-stage Dockerfile

A build of the checked-in multistage file can be invoked with:

```bash
docker build \
  -f Docker-files/app/multistage/Dockerfile \
  -t vprofile-app:multistage \
  .
```

**Important:** This Dockerfile clones `https://github.com/hkhcoder/vprofile-project.git`, checks out its `docker` branch, and builds that external source; it does not compile the contents of this repository. For a reproducible build of this repository’s source, use `Docker-files/app/Dockerfile` after the Maven build unless that external build definition is intentionally desired and verified.

### Database and Nginx image definitions

The database Dockerfile is built with its context set to the database directory because it adds a local SQL initialization file:

```bash
docker build -f Dockerfile -t vprofile-db:local Docker-files/db
```

It declares MySQL 8.0.33, `MYSQL_DATABASE=accounts`, and a root password in the Dockerfile, and adds `db_backup.sql` for initialization. Do not use the checked-in password in a shared or production environment. Provide secrets securely and customize the image configuration before use.

Build the Nginx image using its directory as build context (the config file is copied by relative path):

```bash
docker build -f Dockerfile -t vprofile-web:local Docker-files/web
```

The Nginx config listens on port 80 and proxies to the hostname `vproapp` on port 8080. The backend container must be resolvable by that name on a shared Docker network. For example, on a user-defined network:

```bash
docker network create vprofile-net

docker run -d --name vproapp --network vprofile-net \
  -p 8080:8080 vprofile-app:local

docker run --rm --name vprofile-web --network vprofile-net \
  -p 80:80 vprofile-web:local
```

This starts only the app and proxy containers. It does not configure or start the application’s other dependencies.

## Configuration and environment

`src/main/resources/application.properties` is loaded through Spring's property-placeholder configuration. The checked-in values configure the following properties:

| Property group | Properties in file | Checked-in default / notes |
|---|---|---|
| MySQL | `jdbc.driverClassName`, `jdbc.url`, `jdbc.username`, `jdbc.password` | JDBC URL uses `vprodb:3306/accounts`; user `root` |
| Memcached | `memcached.active.host`, `memcached.active.port`, `memcached.standBy.host`, `memcached.standBy.port` | Active host `vprocache01:11211`; standby `127.0.0.2:11211` |
| RabbitMQ | `rabbitmq.address`, `rabbitmq.port`, `rabbitmq.username`, `rabbitmq.password` | `vpromq01:5672`; development credentials are present in the file |
| Elasticsearch | `elasticsearch.host`, `elasticsearch.port`, `elasticsearch.cluster`, `elasticsearch.node` | `localhost:9300`, cluster `vprofile`, node `vprofilenode` |
| Web and upload | `spring.mvc.view.prefix`, `spring.mvc.view.suffix`, `spring.servlet.multipart.max-file-size`, `spring.servlet.multipart.max-request-size` | JSP views under `/WEB-INF/views/`; upload limits are 128KB |
| Logging and security | `logging.level.*`, `spring.security.user.*` | Debug/security settings and a default user are configured in the properties file |

These are **Spring property keys, not documented environment-variable bindings**. The repository does not show automatic mapping from environment variables to these custom property names. To change values, update the configuration through a supported Spring property source or provide a deliberately configured externalized properties file; verify that the application actually receives those overrides.

Before deployment:

1. Replace fixed service addresses with addresses appropriate to the target network.
2. Remove default credentials from source and image layers; use managed secrets or a secure runtime configuration method.
3. Review the committed SQL backup and logging/security defaults for your intended environment.
4. Confirm service versions, connectivity, database initialization, and compatibility before running the application.

The SQL files `src/main/resources/accountsdb.sql` and `Docker-files/db/db_backup.sql` provide database initialization/backup material. Review the target SQL file before importing it, as database initialization scripts can create schema, insert records, and in some cases drop existing tables.

## CI/CD workflow

The workflow is `.github/workflows/ci.yml` and is triggered by pull requests targeting `main` and pushes to `main`. It selects separate jobs based on the event type.

### Pull requests to `main`

The `build-and-sonar` job runs on `ubuntu-latest` and:

1. Checks out the source with full Git history (`fetch-depth: 0`).
2. Installs Temurin JDK 21 and caches Maven dependencies and the Sonar scanner cache.
3. Runs `mvn clean verify checkstyle:checkstyle -B`.
4. Runs the SonarQube scan and waits for the quality-gate result (polling timeout: 600 seconds).

The job uses the `SONAR_HOST_URL` repository variable and `SONAR_TOKEN` secret.

### Pushes to `main`

The workflow runs two dependent activities:

1. **Build and push image:** Configure AWS credentials; ensure the configured ECR repository exists; build the image from `Docker-files/app/multistage/Dockerfile`; tag it with the first seven characters of the commit SHA and `latest`; authenticate to ECR and push both tags.
2. **Update Helm values in a separate repository:** Clone the Helm repository named by `HELM_REPO_NAME`, update `helm/vprofile/values.yaml` fields `.app.image` and `.app.tag`, then commit and push the change to that repository's `main` branch if the values changed.

The workflow references repository variables `AWS_REGION`, `ECR_REPOSITORY`, `HELM_REPO_NAME`, and `SONAR_HOST_URL`; secrets `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `SONAR_TOKEN`, `HELM_REPO_USER`, and `GITOPS_PAT`. Configure those only with appropriate, least-privilege access. The workflow does not define the values of these variables or secrets.

## SonarQube integration

The repository's `sonar-project.properties` declares:

- Project key and name: `vprofile-app`
- Project version: `1.0`
- Sources: `src/`
- Java binaries: `target/classes`
- JUnit reports: `target/surefire-reports/`
- JaCoCo XML report: `target/site/jacoco/jacoco.xml`
- Checkstyle report: `target/checkstyle-result.xml`

On pull requests, CI runs the Maven verification and Checkstyle report goal before the SonarQube scan and quality-gate action. `pom.xml` has the JaCoCo Maven plugin configuration commented out, so the declared JaCoCo report path does not prove that the report is generated by the current Maven build. Confirm report generation and compatibility with the deployed SonarQube scanner/server versions when configuring or troubleshooting analysis.

## Deployment workflow

The checked-in automation describes a GitOps-style image and configuration handoff, rather than directly deploying to a runtime cluster:

```text
Pull request to main
  └─ Maven verify + Checkstyle report + SonarQube scan/quality gate

Push to main
  ├─ Build app image and push SHA + latest tags to AWS ECR
  └─ Update app.image and app.tag in the Helm repository, then push its main branch
       └─ A separate deployment/reconciliation mechanism would consume that change
```

The workflow itself does not run `helm upgrade`, deploy a cluster, or define the runtime environment. The Helm repository and any environment reconciliation process are separate from this repository and must be configured independently. The image build step currently uses the multistage Dockerfile described above, which builds external repository source rather than this repository checkout; verify or change that build input before relying on the published image as a build of this repository.

## Troubleshooting

| Symptom | Checks |
|---|---|
| Maven cannot resolve dependencies or compile | Confirm Maven has network access to repositories, use a supported JDK (the project compiler targets Java 17), and inspect the first compilation/dependency error. |
| WAR is missing during Docker build | Run `mvn clean package` from the repository root and verify `target/vprofile-v2.war` exists; use the app Dockerfile with repository root as the build context. |
| App starts but database operations fail | Check that MySQL is reachable at the configured JDBC host/port, the `accounts` database and expected tables exist, and credentials match the deployed database. |
| Cache, RabbitMQ, or Elasticsearch operations fail | Check service availability, DNS/network reachability, and the corresponding configured host/port. The repository has no full dependency orchestration file. |
| Nginx returns a gateway error | Confirm the application is running on the same Docker network and is resolvable as `vproapp` on port 8080, matching `nginvproapp.conf`. |
| Sonar scan cannot authenticate/connect | Verify `SONAR_TOKEN` secret and `SONAR_HOST_URL` variable are set, reachable, and valid for the project. Check the quality gate and report paths. |
| ECR push fails | Verify AWS region/repository variables, IAM permissions for ECR and STS, valid AWS credentials, and repository access. |
| Helm update fails | Verify the separate repository name, `HELM_REPO_USER` and `GITOPS_PAT` credentials, write access to its `main` branch, and that `helm/vprofile/values.yaml` contains the expected fields. |

For local diagnosis, inspect Maven output, container logs (`docker logs <container>`), and the GitHub Actions job logs. Avoid printing tokens, passwords, or other secrets in logs.
