# Jenkins Pipeline for Java based application using Maven, SonarQube, Argo CD, Helm and Kubernetes

![Screenshot 2023-03-28 at 9 38 09 PM](https://user-images.githubusercontent.com/43399466/228301952-abc02ca2-9942-4a67-8293-f76647b6f9d8.png)


# Ultimate CI/CD Pipeline — Spring Boot, Jenkins, Docker, Argo CD, Kubernetes

An end-to-end CI/CD pipeline for a Spring Boot application. Jenkins handles
build, test, static code analysis, and Docker image publishing. Argo CD
handles deployment to Kubernetes using GitOps — Jenkins never touches the
cluster directly; it only updates a Git repository, and Argo CD keeps the
cluster in sync with that repository.

## Architecture

```
GitHub (source) → Jenkins → SonarQube → Docker Hub → GitHub (manifests) → Argo CD → Kubernetes
```

1. Code is pushed to this repository
2. Jenkins checks out the code, builds it with Maven, and runs a SonarQube
   code quality scan
3. Jenkins builds a Docker image, tags it with the Jenkins build number,
   and pushes it to Docker Hub
4. Jenkins updates the image tag in the Kubernetes manifests repository
   and commits/pushes that change
5. Argo CD detects the change in the manifests repository and
   automatically syncs the cluster to match
6. Kubernetes performs a rolling update, replacing old pods with new ones
   running the updated image

## Tech stack

- **CI**: Jenkins (Declarative Pipeline), running each build inside a
  Docker container agent with Maven and Docker pre-installed
- **Code quality**: SonarQube
- **Containerization**: Docker
- **Registry**: Docker Hub
- **CD**: Argo CD (GitOps — automated sync, self-healing, pruning enabled)
- **Orchestration**: Kubernetes (tested on Minikube)
- **Application**: Spring Boot (Java 21, Maven)

## Repository structure

```
.
├── Argo CD/
│   └── argocd-basic.yaml
├── spring-boot-app-manifests/
│   ├── deployment.yml
│   └── service.yml
├── spring-boot-app/
│   ├── src/main/
│   ├── .gitignore
│   ├── Dockerfile
│   ├── JenkinsFile
│   ├── README.md
│   └── pom.xml
├── README.md
└── argocd-application.yaml
```

## Pipeline stages (Jenkinsfile)

| Stage | What it does |
|---|---|
| Checkout | Pulls the latest code (handled implicitly by the Jenkins job's SCM configuration) |
| Build and Test | Runs `mvn clean package` — compiles, tests, and packages the application |
| Static Code Analysis | Runs a SonarQube scan against the codebase |
| Build and Push Docker Image | Builds a Docker image tagged with the Jenkins build number and pushes it to Docker Hub |
| Update Deployment File | Updates the image tag in `deployment.yml` in this repository and pushes the commit — this is the handoff point to Argo CD |

## Kubernetes configuration

`deployment.yml` includes:
- Readiness and liveness probes against the app's `/actuator/health` endpoint (via Spring Boot Actuator)
- CPU and memory resource requests/limits

`service.yml` exposes the application via a `NodePort` Service.

## Argo CD

The Argo CD `Application` resource (`argocd-application.yaml`) is
version-controlled in this repository rather than only configured through
the UI. It's configured with:
- `automated.enabled: true` — auto-sync on new commits
- `selfHeal: true` — reverts manual cluster changes back to match Git
- `prune: true` — removes resources deleted from the manifests directory

## Setting this up yourself

1. Fork this repository
2. Configure a Jenkins pipeline job pointing at your fork, using
   "Pipeline script from SCM"
3. Add Jenkins credentials for: Docker Hub (`docker-cred`), SonarQube
   (`sonarqube`), and a GitHub token (`github`) with push access to your fork
4. Update the image name in the `Jenkinsfile` and `deployment.yml` to your
   own Docker Hub namespace
5. Install Argo CD on your cluster and apply `argocd-application.yaml`,
   pointing `repoURL` at your fork
6. Trigger a build

## Notes on scope

This project intentionally focuses on the CI/CD and GitOps mechanics
rather than application complexity — the Spring Boot app itself is a
minimal demo endpoint. Planned/possible extensions not currently
implemented: a manual approval gate before production deploys, an
automated rollback stage, and build/deploy notifications.
