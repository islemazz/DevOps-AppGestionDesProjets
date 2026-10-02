# Gestion des Projets — DevOps CI/CD

A Spring Boot + Angular + MySQL project-management app, containerised with Docker and shipped through two Jenkins pipelines: one that builds, deploys, tests and pushes the images to Docker Hub, and one that analyses the code with SonarQube and blocks the build when the Quality Gate fails.

Built during the DevOps module at ESPRIT School of Engineering (2026–2027). The application code comes from a starter project by Alaa Rami; **my work is the DevOps part**: Dockerfiles, Compose stack, nginx reverse proxy, Jenkins pipelines, SonarQube integration and the security fix.

## Architecture

```
GitHub ──► Jenkins (Ubuntu 22.04 VM, Vagrant/VirtualBox)
              ├─ gestion-pipeline   : docker compose build → up → smoke test → push to Docker Hub
              └─ Pipeline-SonarQube : compile → tests (H2) → SonarQube analysis → Quality Gate → package

Docker Compose stack (network gestion-net)
   browser ──► frontend (nginx :8083) ──/entreprise, /equipe, /projet…──► backend (Spring Boot :8080) ──► mysql (8.0, volume db-data)
```

Only nginx is exposed. The Angular app calls the API with relative URLs, and nginx proxies them to the backend, so frontend and API share one origin (no CORS needed).

## Stack

| Layer | Technology |
| --- | --- |
| Frontend | Angular, served by nginx:alpine (multi-stage build from node:24-alpine) |
| Backend | Spring Boot 4, Java 17 (multi-stage build: maven:3.9 → eclipse-temurin:17-jre-alpine) |
| Database | MySQL 8.0 with healthcheck and named volume; H2 in-memory for tests |
| CI/CD | Jenkins (declarative pipelines, credentials, Blue Ocean / Pipeline Graph View) |
| Code quality | SonarQube Community 26.9 (Docker), webhook back to Jenkins |
| Registry | Docker Hub: `islemaz/gestion-backend`, `islemaz/gestion-frontend` |
| Infra | Vagrant + VirtualBox, Ubuntu 22.04 VM (4 GB RAM, 1 vCPU) |

## Run it locally

```bash
git clone https://github.com/islemazz/DevOps-AppGestionDesProjets.git
cd DevOps-AppGestionDesProjets
docker compose up -d --build
# open http://localhost:8083
```

## Pipeline 1 — build, deploy, push (`Jenkinsfile`)

1. **Checkout** from GitHub
2. **Build images** with `docker compose build`
3. **Deploy** with `docker compose up -d`
4. **Smoke test**: `curl /entreprise/all` until the API answers (30 tries × 5 s)
5. **Push** both images to Docker Hub, tagged with the build number and `latest` (Docker Hub login stored as a Jenkins credential, never in the repo)

## Pipeline 2 — SonarQube Quality Gate

GIT → Build (`mvn compile`) → Tests (`mvn test` on H2) → SonarQube (`withSonarQubeEnv`) → Quality Gate (`waitForQualityGate`, aborts on failure) → Package (`mvn package`)

The SonarQube token lives only in Jenkins Credentials, never in the repo.

### Results

| Metric | First analysis | After the CORS fix |
| --- | --- | --- |
| Quality Gate | Passed | Passed |
| Security issues | 12 (rating D) | 8 (rating D) |
| Bugs / code smells | 0 / 0 (A / A) | 0 / 0 (A / A) |
| Duplications | 0.0 % | 0.0 % |
| Technical debt | 2d 1h | 1h 20min |
| Pipeline time | 11 min 42 s | 2 min 42 s |

The fix removed `@CrossOrigin("*")` from the 4 controllers: it allowed any website to call the API, and it was no longer needed behind the nginx proxy. The 8 remaining issues ask for DTOs instead of JPA entities in `@RequestBody` (mass assignment, CWE-915). That fix is the next step, along with JaCoCo for test coverage.

## Problems solved

| Problem | Fix |
| --- | --- |
| Frontend called `localhost:8080`, which in the browser is the user's PC | Relative API URLs + nginx reverse proxy |
| Backend looked for MySQL on `localhost` inside its container | `SPRING_DATASOURCE_URL` pointing to the `mysql` service |
| Backend started before MySQL was ready | Healthcheck + `condition: service_healthy` + `restart: on-failure` |
| Tests needed a live MySQL in the pipeline | H2 in-memory database in `src/test/resources` |
| Docker pulls failed over IPv6 on the VM network | IPv6 disabled in the VM |
| SonarQube's embedded Elasticsearch needs a higher kernel limit than Ubuntu's default (65530) | `vm.max_map_count=524288` in `/etc/sysctl.d` |

## Author

Islem Azzouz, Cloud Computing engineering student at ESPRIT, Tunisia.
