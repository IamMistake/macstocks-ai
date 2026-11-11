# MacStocks AI

Macedonian stock-market analytics with a Python data pipeline, a Spring Boot API and a React dashboard. Includes Docker builds, GitHub Actions CI/CD and Kubernetes deployment configuration for Azure Kubernetes Service (AKS).

Originally developed as a university software design and architecture project, previously named `DASproject`.

[Features](#features) · [Run locally](#run-locally) · [API examples](#api-examples) · [CI/CD and Kubernetes](#cicd-and-kubernetes)

## Features

- Collect historical prices and trading volumes from the [Macedonian Stock Exchange](https://www.mse.mk/) through a pipe-and-filter scraping pipeline.
- Backfill up to ten years of history and fetch missing records incrementally, storing results in PostgreSQL and deduplicating by issuer and date.
- Explore prices and technical indicators in a React dashboard, including moving averages, RSI, MACD, stochastic oscillator, ADX and CCI.
- Include issuer-news scraping utilities and rule-based sentiment classification. The sentiment component uses word lists, not a trained ML model or an LLM.
- Build and publish separate Docker images for the backend, frontend and Python scraper, with Kubernetes manifests for deployment and persistent database storage.

> [!NOTE]
> This is an educational analytics project, not a trading platform or investment recommendation service. The dashboard news endpoint currently returns a fixed response rather than the news-scraping results.

## How it works

The Python pipeline discovers issuer codes, downloads historical records, normalizes Macedonian number/date formats and fills data gaps. The Spring Boot API queries PostgreSQL and calculates indicators. The React/TypeScript frontend displays the results with Chart.js.

| Component | Location | Tools |
|---|---|---|
| Data ingestion | [`ScrapingService/`](ScrapingService/) | Python, Requests, Beautiful Soup, psycopg2 |
| REST API and analytics | [`DesignAndArchitectureProject/`](DesignAndArchitectureProject/) | Java 17, Spring Boot 3.4, Spring Data JPA |
| Dashboard | [`frontend/`](frontend/) | React 19, TypeScript, Vite, Chart.js, Tailwind CSS |
| Local services | [`docker-compose.yml`](docker-compose.yml) | PostgreSQL 15, Docker Compose |
| Delivery | [Workflows](.github/workflows/) and [manifests](kubernetes/) | GitHub Actions, Docker Hub, Kubernetes, Azure AKS |

## Run locally

You need Git, Docker with Compose, and Node.js 22+ with npm. The commands below run the scraper and backend in containers and the frontend through Vite. They use the repository's local-development database settings.

```bash
git clone https://github.com/IamMistake/macstocks-ai.git
cd macstocks-ai
docker compose up -d db
docker compose build app backend
```

### Load stock data

```bash
docker compose run --rm app
```

The scraper creates the ingestion tables before fetching data. It currently processes the first 25 discovered issuer codes. Fetching depends on the exchange's availability and HTML structure; avoid repeatedly running full backfills against the source site.

### Start the API

In another terminal:

```bash
docker compose run --rm --service-ports \
  -e SPRING_JPA_HIBERNATE_DDL_AUTO=update backend
```

The API listens on `http://localhost:8080`. The `update` override allows Hibernate to reconcile the local database schema; use it only with a disposable development database. The checked-in backend configuration otherwise uses schema validation.

### Start the dashboard

In [`frontend/src/services/api.ts`](frontend/src/services/api.ts), change `API_BASE_URL` from the checked-in deployment address to:

```typescript
const API_BASE_URL = "http://localhost:8080/api/stocks";
```

Then run:

```bash
cd frontend
npm install
npm run dev
```

Open the local URL printed by Vite, normally `http://localhost:5173`.

> [!IMPORTANT]
> The checked-in configuration needs adjustment for each environment. In particular, the frontend Dockerfile serves Nginx on port 80, while Compose maps port 5173. Use Vite as above for local frontend development, or correct the container port mapping. Check the Nginx proxy's path handling when routing requests to `/api/stocks`.

### Development checks

```bash
# From frontend/
npm run lint
npm run build

# From DesignAndArchitectureProject/; requires JDK 17+ and a configured database
bash ./mvnw test
```

Stop the local database with `docker compose stop db` from the repository root. The named volume retains its data.

## API examples

With the backend running and the selected issuer loaded:

```bash
curl 'http://localhost:8080/api/stocks/companies'
curl 'http://localhost:8080/api/stocks/dashboard/stats?companySelected=ALK'
curl 'http://localhost:8080/api/stocks/dashboard/lineChart?companySelected=ALK&fromYear=2020&toYear=2024'
curl 'http://localhost:8080/api/stocks/dashboard/statistics?companySelected=ALK&fromYear=2020&toYear=2024'
curl 'http://localhost:8080/api/stocks/dashboard/news?companySelected=ALK'
```

The company selector currently returns `KMB`, `ALK` and `SBT`. Choose a symbol present in your database; the selector is not a dynamic list of all scraped issuers.

## CI/CD and Kubernetes

| Workflow | Trigger | What it does |
|---|---|---|
| [`linting.yml`](.github/workflows/linting.yml) | Push to `main`/`cicd`; PR opened | Runs frontend ESLint |
| [`docker.yml`](.github/workflows/docker.yml) | Push to `main`/`cicd` | Builds and pushes backend, frontend and scraper images to Docker Hub |
| [`cd-part.yml`](.github/workflows/cd-part.yml) | Push to `main` | Authenticates to Azure and applies Kubernetes manifests to AKS |
| [`sonarcloud.yml`](.github/workflows/sonarcloud.yml) | Push/PR to `prod` | Runs the scraper, Maven verification and SonarCloud analysis |

Configure `DOCKER_USERNAME` and `DOCKER_TOKEN` for image publishing, `AZURE_CREDENTIALS` for AKS deployment, and `SONAR_TOKEN` and `TOKEN_GITHUB` for the analysis workflow. Use GitHub Actions secrets, not credentials committed to source control.

The [`kubernetes/`](kubernetes/) directory defines the `dians` namespace, backend/frontend deployments and services, PostgreSQL with persistent storage, a scraper job and ingress routing. Before deploying, replace the example database credentials, Docker image references, Azure resource-group/cluster names and ingress host with your own values. Keep the scraper's Python `DB_*` environment variables consistent with `ScrapingService/config.py`; its current Kubernetes job instead supplies Spring datasource variables.

Image publishing and AKS deployment are separate workflows. They are not sequenced behind one another and currently use mutable `latest` tags, so do not assume deployment waits for the latest image build.

## Setup notes

- Existing credentials and infrastructure addresses are development examples. Replace them before exposing any service; rotate any checked-in credential that was used outside a disposable environment.
- The scraper has date-handling and completion-timing code that can raise errors. Inspect its logs and verify stored data before relying on a completed import.
- Backend container builds use Java 18 base images, while Maven targets Java 17. The frontend container uses Node 18. Review image availability and supported runtime versions for your environment.
- The application and cloud deployment were not exercised as part of this README update. Configuration examples describe the repository, not a verified live service.
