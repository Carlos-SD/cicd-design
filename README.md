# CICD-DEMO

A CI/CD demonstration project using Jenkins, SonarQube, and Trivy.  
Spring Boot application with a complete declarative pipeline: build, static analysis, container security scanning, and local deployment.

---

## Pipeline Architecture

```
Git Push
   │
   ▼
┌─────────────┐
│  Checkout   │  Clones the repository from SCM
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Build & Test   │  mvn clean package  (unit tests + JaCoCo coverage)
└──────┬──────────┘
       │
       ▼
┌──────────────┐
│ Docker Build │  docker build -t mi-app:latest .
└──────┬───────┘
       │
       ▼
┌──────────────────────────┐
│ Static Analysis          │  mvn sonar:sonar → SonarQube
│ (SonarQube)              │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ Quality Gate             │  Fails if:
│                          │  • QG status != OK
│                          │  • Security Hotspots pending review > 0
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│ Container Security Scan  │  trivy image --severity CRITICAL
│ (Trivy)                  │  Fails if any CRITICAL vulnerability found
└──────┬───────────────────┘
       │
       ▼
┌─────────────┐   (master branch only)
│   Deploy    │  docker run -d -p 80:8080 mi-app:latest
└─────────────┘
```

---

## Tech Stack

| Component        | Version        | Role                                        |
|------------------|----------------|---------------------------------------------|
| Spring Boot      | 2.7.18         | Java application                            |
| Java             | 21 (JRE Alpine)| Container runtime                           |
| Maven            | 3.9+           | Build and dependency management             |
| JaCoCo           | 0.8.11         | Code coverage                               |
| JUnit            | 4.13.2         | Unit and integration tests                  |
| Jenkins          | LTS            | CI/CD server                                |
| SonarQube        | LTS Community  | Static analysis and Quality Gate            |
| Trivy            | 0.70+          | Docker image vulnerability scanning         |
| Tomcat (embedded)| 9.0.116        | Embedded web server (no CRITICAL CVEs)      |
| PostgreSQL       | 13 Alpine      | SonarQube database                          |

---

## Prerequisites

| Tool    | Min Version | Notes                                                        |
|---------|-------------|--------------------------------------------------------------|
| Docker  | 20+         | Required for all steps                                       |
| Jenkins | LTS         | Plugins: Git, Pipeline, SonarQube Scanner, Docker Pipeline   |
| Trivy   | 0.70+       | Installed inside the Jenkins container                       |

---

## Local Infrastructure Setup

### 1. SonarQube + Database

```bash
docker-compose up -d sonarqube-db sonarqube
```

Access `http://localhost:9000` with `admin / admin` (or your configured password).  
Create a project with key `cicd-demo` and generate an authentication token (type *Global Analysis Token*).

#### Webhook for Quality Gate

In SonarQube → **Administration → Configuration → Webhooks → Create**:

| Field | Value                                    |
|-------|------------------------------------------|
| Name  | Jenkins                                  |
| URL   | `http://jenkins:8080/sonarqube-webhook/` |

> This allows `waitForQualityGate()` to receive the callback instead of timing out in PENDING state.

### 2. Jenkins

```bash
docker run -d \
  --name jenkins \
  --network cicd-design_default \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

> The `cicd-design_default` network is created by docker-compose. This lets Jenkins resolve `sonarqube:9000` by service name.

Unlock Jenkins at `http://localhost:8080` and install the suggested plugins plus:
- **SonarQube Scanner**
- **Docker Pipeline**

#### Additional tools inside the Jenkins container

```bash
docker exec -u root jenkins bash -c "
  # Maven
  apt-get update && apt-get install -y maven

  # Docker CLI
  curl -fsSL https://get.docker.com | sh

  # Trivy
  curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
    | sh -s -- -b /usr/local/bin
"
```

### 3. Jenkins Configuration

#### SonarQube Token Credential

**Manage Jenkins → Credentials → Global → Add Credential**:

| Field  | Value                          |
|--------|--------------------------------|
| Kind   | Secret text                    |
| ID     | `SONAR_AUTH_TOKEN`             |
| Secret | Token generated in SonarQube   |

#### SonarQube Server

**Manage Jenkins → Configure System → SonarQube servers**:

| Field | Value                   |
|-------|-------------------------|
| Name  | `SonarQube`             |
| URL   | `http://sonarqube:9000` |
| Token | `SONAR_AUTH_TOKEN`      |

#### Create the Pipeline Job

1. New item → type **Pipeline**.
2. *Pipeline Definition* → **Pipeline script from SCM**.
3. SCM: Git → repository URL.
4. Script Path: `Jenkinsfile`.

---

## Quality Gates

The pipeline fails automatically in the following scenarios:

| Tool       | Failure condition                                              |
|------------|----------------------------------------------------------------|
| SonarQube  | Quality Gate status is not `OK`                               |
| SonarQube  | At least 1 Security Hotspot with status `TO_REVIEW`           |
| Trivy      | At least 1 `CRITICAL` vulnerability found in the Docker image |

### `.trivyignore` file

The `.trivyignore` file at the project root suppresses explicitly accepted CVEs with justification:

```
# CVE-2016-1000027: HttpInvoker is not used in this application.
# Only fixed in Spring 6.0+ (Spring Boot 3.x); migration deferred.
CVE-2016-1000027
```

---

## Project Structure

```
cicd-design/
├── Jenkinsfile            # Complete declarative pipeline (7 stages)
├── Dockerfile             # Docker image: eclipse-temurin:21-jre-alpine
├── .trivyignore           # Accepted CVEs with justification
├── docker-compose.yml     # SonarQube + PostgreSQL + builder + selenium
├── pom.xml                # Spring Boot 2.7.18, JaCoCo 0.8.11, Tomcat 9.0.116
├── src/
│   ├── main/              # Spring Boot source code
│   └── test/              # JUnit 4 tests (Unit, Integration, System categories)
└── k8s-config/            # Kubernetes manifests (cluster deployment)
```

---

## Testing the Pipeline

```bash
# Make a code change, then commit and push
# Jenkins detects the change and runs the pipeline automatically
git add .
git commit -m "test: trigger pipeline"
git push origin master
```

If the code passes all quality gates, the application is deployed at `http://localhost:80/api`.

---

## `post` Block and Notifications

The pipeline cleans the workspace on completion (success or failure) via `cleanWs()`.  
To enable email notifications on failure, uncomment in `Jenkinsfile`:

```groovy
mail to: 'team@example.com',
     subject: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
     body: "See logs at: ${env.BUILD_URL}"
```

Then configure the SMTP server in **Manage Jenkins → Configure System → E-mail Notification**.
