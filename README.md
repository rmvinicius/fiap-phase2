# ToggleMaster — Tech Challenge FIAP (Fase 2)

Plataforma de **Feature Flags** (alternância de funcionalidades) baseada em microsserviços, com três formas de execução:

- **docker** — orquestração local via `docker-compose`.
- **k8s** — execução local em cluster **Minikube** (Kubernetes).
- **eks** — implantação em produção na **AWS** (EKS + serviços gerenciados).

---

## Visão Geral

O ToggleMaster permite criar, gerenciar e avaliar *feature flags* de forma segura e escalável. O fluxo de dados é dividido em:

- **Caminho administrativo (cold path):** criação de chaves de API, definição de flags e regras de segmentação.
- **Caminho quente (hot path):** avaliação da flag por parte dos clientes finais, otimizada com cache em Redis e envio assíncrono de eventos para uma fila.
- **Caminho analítico:** um *worker* consome a fila e persiste os eventos em um banco NoSQL para análise.

### Fluxo de ponta a ponta

```
auth-service ──valida chave──▶ flag-service / targeting-service
                                        │
evaluation-service ◀── busca flag+regra (cache Redis) ──┘
        │
        ├─ retorna true/false ao cliente (hot path)
        └─ publica EvaluationEvent na fila SQS/ElasticMQ
                                        │
analytics-service ◀── consome fila ─────┘
        │
        └─ grava evento no DynamoDB (analytics)
```

### Tecnologias

| Camada | Tecnologia |
| ------ | ---------- |
| Linguagens | Go (auth, evaluation), Python/Flask (flag, targeting, analytics) |
| Banco relacional | PostgreSQL 17 (uma base por serviço) |
| Cache | Redis 7.2 |
| Fila (mensageria) | AWS SQS (nuvem) / ElasticMQ (local, compatível com SQS) |
| Banco analítico | AWS DynamoDB (nuvem) / DynamoDB Local (local) |
| Orquestração | Docker Compose / Kubernetes (Minikube) / AWS EKS |
| Ingress | NGINX Ingress Controller (Helm) |
| Escalabilidade | HPA (CPU/memória) + KEDA (escala orientada por fila SQS) |
| Registry (AWS) | Amazon ECR |

---

## Microserviços

### 1. `auth-service` (Go)
Serviço de **autenticação**. Cria e valida chaves de API (`tm_key_...`) usadas por todos os demais serviços.
- Banco: `auth_db` (PostgreSQL)
- Porta: `8001`
- Principais endpoints:
  - `GET /health` — health check
  - `POST /admin/keys` — cria chave de API (requer `MASTER_KEY` no `Authorization: Bearer`)
  - `GET /validate` — valida a chave informada no header `Authorization`

### 2. `flag-service` (Python/Flask)
CRUD das **definições** das feature flags. Protegido — toda requisição (exceto `/health`) exige `Authorization: Bearer <api-key>`.
- Banco: `flags_db` (PostgreSQL)
- Porta: `8002`
- Dependência: `auth-service`
- Principais endpoints:
  - `GET /health`
  - `GET/POST /flags` — lista / cria flag
  - `PUT /flags/<name>` — atualiza flag

### 3. `targeting-service` (Python/Flask)
Gerencia **regras de segmentação** (ex.: "50% dos usuários", "país X") de uma flag específica. Protegido por API key.
- Banco: `targeting_db` (PostgreSQL)
- Porta: `8003`
- Dependência: `auth-service`
- Principais endpoints:
  - `GET /health`
  - `GET/POST /rules` — busca / cria regra
  - `PUT /rules/<flag_name>` — atualiza regra (`type: PERCENTAGE`, `value: 0-100`)

### 4. `evaluation-service` (Go)
O **hot path**: único endpoint chamado pelos clientes finais. Avalia a flag para um `user_id` usando cache Redis (TTL curto) e publica o evento da decisão na fila **SQS** de forma assíncrona.
- Porta: `8004`
- Dependências: `auth-service`, `flag-service`, `targeting-service`, `redis`, fila (SQS/ElasticMQ)
- Endpoint:
  - `GET /health`
  - `GET /evaluate?user_id=...&flag_name=...` → `{"flag_name":..., "user_id":..., "result": true|false}`
- Escalabilidade: **HPA** por CPU e memória (1–5 réplicas).

### 5. `analytics-service` (Python/Flask)
*Worker* de **análise** (sem API pública além de `/health`). Consome continuamente a fila SQS e grava os eventos na tabela `ToggleMasterAnalytics` do **DynamoDB**.
- Porta: `8005` (somente health check)
- Dependências: fila SQS, DynamoDB
- Tabela DynamoDB esperada: `ToggleMasterAnalytics` (partition key `event_id` do tipo String)
- Escalabilidade: **KEDA** orientado pelo comprimento da fila SQS (0–5 réplicas, `queueLength: 5`).

---

## Infraestrutura de Suporte

| Componente | Docker | k8s (Minikube) | EKS (AWS) |
| ---------- | ------ | -------------- | --------- |
| PostgreSQL (auth/flags/targeting) | contêiner `custom-postgres:17` | Stateful/Deployment + PV/PVC por base | **Amazon RDS** (3 instâncias) |
| Redis (cache) | contêiner `custom-redis:7.2` | `StatefulSet` + PVC | **ElastiCache (Redis OSS)** |
| DynamoDB (analytics) | `custom-dynamodb:3.3.0` (local) | Deployment + PVC | **Amazon DynamoDB** (tabela gerenciada) |
| Fila SQS | `custom-elasticmq:1.7.1` (compatível SQS) | Deployment + PVC | **Amazon SQS** (fila gerenciada) |

> No EKS, os componentes de dados são **serviços gerenciados da AWS**; por isso a pasta `eks/` contém apenas os manifestos de *namespace*, *ingress* e *keda*, enquanto os manifestos por serviço ficam em `ms/<service>/eks/`.

---

## Estrutura de Pastas

### `docker/` — Docker Compose (local)
Imagens customizadas construídas a partir dos `Dockerfile`s auxiliares e do `docker-compose.yaml` que sobe todos os serviços e dependências em uma rede bridge `fase2`.

```
docker/
├── docker-compose.yaml        # orquestra todos os serviços + dependências
├── postgres/
│   └── Dockerfile             # custom-postgres:17
├── redis/
│   └── Dockerfile             # custom-redis:7.2
├── dynamodb/
│   └── Dockerfile             # custom-dynamodb:3.3.0
└── sqs/
    └── Dockerfile             # custom-elasticmq:1.7.1
```

Serviços e portas expostas no compose: `auth-service:8001`, `flag-service:8002`, `targeting-service:8003`, `evaluation-service:8004`, `analytics-service:8005`, `dynamodb:8000`, `elasticmq:9324/9325`, `redis:6379`, além dos Postgres em `5439/5440/5441`.

### `k8s/` — Minikube (Kubernetes local)
Manifestos divididos por componente/namespace. Usa `StorageClass` + PV/PVC locais (`minikube mount`) para persistência.

```
k8s/
├── namespace/namespace.yaml        # namespaces: postgres, redis, dynamodb, elasticmq, application
├── storageclass/storageclass.yaml  # StorageClass local para PVs
├── postgres/
│   ├── auth-service-db/            # configmap, secret, deployment, pv, pvc
│   ├── flag-service-db/
│   └── targeting-service-db/
├── redis/
│   ├── configmap.yaml              # redis.conf
│   ├── stateful.yaml               # StatefulSet redis
│   ├── pv.yaml / pvc.yaml
│   └── service.yaml
├── dynamodb/
│   ├── deployment.yaml / service.yaml / pv.yaml / pvc.yaml
├── elasticmq/
│   ├── deployment.yaml / service.yaml / pv.yaml / pvc.yaml
├── ingress/
│   ├── ingress.md                  # comandos helm do ingress-nginx
│   ├── values.yaml                 # valores do chart ingress-nginx
│   ├── ingress.yaml                # roteamento /auth /flags /targeting /evaluation /analytics
│   └── route.yaml                  # Services ExternalName ponteando para o namespace application
├── keda/
│   └── keda.md                     # instalação do KEDA via Helm
├── metrics/
│   └── metrics.yaml                # metrics-server (necessário p/ HPA por recurso)
└── minikube.md                     # comandos de start, load de imagens e mount
```

### `eks/` — AWS EKS (produção)
Apenas os recursos de cluster que não são gerenciados pela AWS. Os deployments por serviço ficam em `ms/<service>/eks/`.

```
eks/
├── namespace/namespace.yaml        # mesmos namespaces do k8s
├── ingress/
│   ├── ingress.md                  # instalação ingress-nginx (Helm)
│   ├── values.yaml
│   ├── ingress.yaml                # mesmo roteamento de paths do k8s
│   └── route.yaml                  # ExternalName services
└── keda/
    └── keda.md                     # instalação KEDA (Helm)
```

### `ms/` — Código e manifestos dos microsserviços
Cada serviço contém seu código-fonte, `Dockerfile`, `.env` de exemplo e duas subpastas de manifestos: `k8s/` (Minikube) e `eks/` (AWS).

```
ms/
├── auth-service/         (Go)
│   ├── main.go, handlers.go, key.go, go.mod
│   ├── db/init.sql
│   ├── k8s/   (configmap, secret, deployment, service)
│   └── eks/   (configmap, secret, deployment, service)
├── flag-service/         (Python)
│   ├── app.py, requirements.txt, db/init.sql
│   ├── k8s/   (configmap, secret, deployment, service)
│   └── eks/   (configmap, secret, deployment, service)
├── targeting-service/    (Python)
│   ├── app.py, requirements.txt, db/init.sql
│   ├── k8s/   (configmap, secret, deployment, service)
│   └── eks/   (configmap, secret, deployment, service)
├── evaluation-service/   (Go)
│   ├── main.go, handlers.go, evaluator.go, sqs.go, types.go
│   ├── k8s/   (configmap, secret, deployment, service, hpa)
│   └── eks/   (configmap, secret, deployment, service, hpa)
└── analytics-service/    (Python)
    ├── app.py, requirements.txt
    ├── k8s/   (configmap, secret, deployment, service, keda)
    └── eks/   (configmap, secret, deployment, service, keda)
```

**Diferenças k8s × eks nos manifestos de serviço:**
- **Imagem:** no `k8s/` usa imagens locais (`auth-service:1.0`); no `eks/` usa imagens do ECR (`076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/<svc>:1.0`).
- **Endpoints de nuvem:** no `eks/` as URLs de SQS/DynamoDB/Redis apontam para os serviços gerenciados da AWS; no `k8s/` apontam para os serviços internos do cluster (ex.: `elasticmq-svc.elasticmq.svc.cluster.local:9324`).
- **Segredos AWS:** no `eks/` os Secrets incluem credenciais reais da AWS (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`); no `k8s/` usam credenciais locais (`local`).

---

## Escalabilidade

- **`evaluation-service` — HPA:** escala de 1 a 5 réplicas com base em **CPU (80%)** e **memória (80%)**. Requer o `metrics-server` (instalado pelo `k8s/metrics/metrics.yaml`).
- **`analytics-service` — KEDA:** `ScaledObject` tipo `aws-sqs-queue` escala de **0 a 5 réplicas** conforme o número de mensagens na fila (`queueLength: 5`, `pollingInterval: 10s`, `cooldownPeriod: 30s`). Usa `TriggerAuthentication` referenciando um Secret com as credenciais AWS.

---

## Como Executar

### 1. Docker Compose
Na pasta `docker/`, após construir as imagens customizadas (`postgres`, `redis`, `dynamodb`, `sqs`) e as imagens dos serviços:

```bash
cd docker
docker compose up -d
```

### 2. Kubernetes local (Minikube)
Consulte `k8s/minikube.md`:
```bash
minikube start --driver=docker --network=minikube-custom
# carregar imagens
minikube image load custom-postgres:17 custom-redis:7.2 custom-dynamodb:3.3.0 \
  custom-elasticmq:1.7.1 auth-service:1.0 flag-service:1.0 \
  targeting-service:1.0 evaluation-service:1.0 analytics-service:1.0
# montar volumes para PVs
minikube mount /home/vinicius.mendes/Kubernetes/volumes/:/mnt/volumes
```

Aplicar na ordem: namespaces → storageclass → postgres → redis → dynamodb → elasticmq → metrics-server → deployments/services de cada serviço → ingress (Helm) → keda (Helm) + ScaledObjects/HPA.

### 3. AWS EKS
Provisionar a infra na AWS (VPC/subnets, RDS, ElastiCache, DynamoDB, SQS, ECR, EKS) conforme `TODO.md`. Build/push das imagens para o ECR e aplicar:
```bash
aws eks update-kubeconfig --region us-east-1 --name eks-dev
# secrets de pull do ECR
kubectl create secret docker-registry ecr-secret \
  --docker-server=076892642827.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1)
# namespaces, ingress, keda e os manifestos em ms/<svc>/eks/
kubectl apply -f eks/namespace/namespace.yaml
kubectl apply -f ms/auth-service/eks/
# ... demais serviços
```

---

## Variáveis de Ambiente (resumo)

| Serviço | Variáveis principais |
| ------- | -------------------- |
| auth-service | `DATABASE_URL`, `PORT`, `MASTER_KEY` |
| flag-service | `DATABASE_URL`, `PORT`, `AUTH_SERVICE_URL` |
| targeting-service | `DATABASE_URL`, `PORT`, `AUTH_SERVICE_URL` |
| evaluation-service | `PORT`, `REDIS_URL`, `FLAG_SERVICE_URL`, `TARGETING_SERVICE_URL`, `SERVICE_API_KEY`, `AWS_SQS_URL`, `AWS_SQS_ENDPOINT`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |
| analytics-service | `PORT`, `AWS_SQS_URL`, `AWS_SQS_ENDPOINT`, `AWS_DYNAMODB_TABLE`, `AWS_DYNAMODB_ENDPOINT`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` |

Consulte os `README.md` de cada serviço em `ms/<service>/README.md` para exemplos completos e testes de endpoints.

---

## Roteamento (Ingress)

O NGINX Ingress expõe os serviços pela mesma porta, com *rewrite* de path:

| Path | Serviço | Porta |
| ---- | ------- | ----- |
| `/auth` | auth-service | 8001 |
| `/flags` | flag-service | 8002 |
| `/targeting` | targeting-service | 8003 |
| `/evaluation` | evaluation-service | 8004 |
| `/analytics` | analytics-service | 8005 |
