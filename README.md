<p align="right"><b>EN</b> | <a href="README.fr.md">FR</a></p>

# Spring Microservices with HashiCorp Vault

Externalizing the secrets of Spring Boot microservices into HashiCorp Vault: no credentials in the code or in configuration files, they are fetched from Vault at startup. The stack is deployed with Docker Compose through a Jenkins pipeline and monitored with Prometheus, Loki, Tempo and Grafana.

![Java](https://img.shields.io/badge/Java-17-007396?style=flat-square&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.2-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![Spring Cloud Vault](https://img.shields.io/badge/Spring_Cloud_Vault-2023.0-6DB33F?style=flat-square&logo=spring&logoColor=white)
![Vault](https://img.shields.io/badge/HashiCorp_Vault-KV-FFEC6E?style=flat-square&logo=vault&logoColor=black)
![Docker](https://img.shields.io/badge/Docker_Compose-2496ED?style=flat-square&logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?style=flat-square&logo=jenkins&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-observability-F46800?style=flat-square&logo=grafana&logoColor=white)

## Architecture

```mermaid
flowchart LR
    V[(HashiCorp Vault<br/>KV secret engine)]
    C[customer-app<br/>:8075] -->|startup: read secret/customer-app| V
    I[inventory-service<br/>:8098] -->|startup: read secret/inventory-service| V
    U[User] -->|OAuth2 login<br/>Google / GitHub| C
    C -->|Bearer JWT| I

    C & I -->|metrics| P[Prometheus]
    C & I -->|traces| T[Tempo]
    C & I -->|logs| L[Loki]
    P & T & L --> G[Grafana]
```

| Component | Role |
|---|---|
| **customer-app** | Thymeleaf web app, OAuth2 client (Google, GitHub). Its client IDs and secrets live in Vault. |
| **inventory-service** | REST API secured as an OAuth2 resource server; its configuration also comes from Vault. |
| **Vault** | Stores each service's secrets under `secret/<application-name>`. |
| **Observability** | Prometheus (metrics), Tempo (traces), Loki (logs), cAdvisor (containers) and Grafana with linked data sources: a trace can be opened from a log line and the other way around. |

## How secrets are loaded

Each service uses `spring-cloud-starter-vault-config`. At startup, it authenticates to Vault with the token passed in `VAULT_TOKEN`, reads the KV path named after the application, and injects the values as regular Spring properties.

```text
secret/customer-app
├── spring.security.oauth2.client.registration.google.clientId
├── spring.security.oauth2.client.registration.google.client-secret
├── spring.security.oauth2.client.registration.github.clientId
└── spring.security.oauth2.client.registration.github.client-secret
```

The code and `application.properties` contain no secret: changing a credential only requires updating Vault and restarting the service.

## CI/CD

The [jenkinsfile](jenkinsfile) checks out the repository and redeploys the whole stack with `docker compose up --build`. The Vault address and token come from Jenkins credentials (`vault-addr`, `vault-token`), never from the repository.

## Getting started

1. Start Vault and write the secrets:

   ```bash
   docker network create vault_network
   docker run -d --name vault --network vault_network -p 8200:8200 \
     -e VAULT_DEV_ROOT_TOKEN_ID=<token> hashicorp/vault

   export VAULT_ADDR=http://localhost:8200 VAULT_TOKEN=<token>
   vault kv put secret/customer-app \
     spring.security.oauth2.client.registration.google.clientId=... \
     spring.security.oauth2.client.registration.google.client-secret=... \
     spring.security.oauth2.client.registration.github.clientId=... \
     spring.security.oauth2.client.registration.github.client-secret=...
   ```

2. Start the stack:

   ```bash
   export VAULT_TOKEN=<token> GRAFANA_ADMIN_PASSWORD=<password>
   docker compose up --build -d
   ```

| Service | URL |
|---|---|
| customer-app | http://localhost:8075 |
| inventory-service | http://localhost:8098 |
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| cAdvisor | http://localhost:8070 |
