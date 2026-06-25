### auth-service

# Desafios:

1- Problema para realizar o a execução do go build
  - Necessário comentar no go.mod algumas linhas, handlers.go e main.go

## Criação de uma database local

# Build do postgres

cd ./auth-service/db/Dockerfile
docker build -t auth-service-postgres:17 .

# docker compose

services:
  auth-service-db:
    image: auth-service-postgres:17
    container_name: auth-service-db
    restart: always
    environment:
      POSTGRES_USER: auth_service
      POSTGRES_PASSWORD: auth_service@2026
      POSTGRES_DB: auth_db
    ports:
      - "5440:5432"
    volumes:
      - ./volume/postgres_data:/var/lib/postgresql/data

# Acessar

psql -h localhost -p 5440 -U auth_service -d auth_db

# Executar sql

psql -h localhost -p 5440 -U auth_service -d auth_db -f ./db/init.sql

## Build da imagem do serviço auth-service

docker build -t auth-service:1.0 .

==============================

### flag-service

# desafios

Necessário adicionar Werkzeug==2.2.3 no requirements.txt, pois esta falhando para iniciar a aplicação.

## Criação de uma database local

# Build do postgres

cd ./flag-service/db/Dockerfile
docker build -t flag-service-postgres:17 .

## Build da imagem do serviço flag-service

docker build -t flag-service:1.0 .

==============================

### targeting-service

# desafios

Erro ao rodar:

  File "/app/app.py", line 8, in <module>
    from flask import Flask, request, jsonify
  File "/opt/venv/lib/python3.11/site-packages/flask/__init__.py", line 5, in <module>
    from .app import Flask as Flask
  File "/opt/venv/lib/python3.11/site-packages/flask/app.py", line 30, in <module>
    from werkzeug.urls import url_quote
ImportError: cannot import name 'url_quote' from 'werkzeug.urls' (/opt/venv/lib/python3.11/site-packages/werkzeug/urls.py)

Necessário adicionar Werkzeug==2.2.3 no requirements.txt, pois esta falhando para iniciar a aplicação.

## Criação de uma database local

# Build do postgres

cd ./targeting-service/db/Dockerfile
docker build -t targeting-service-postgres:17 .

## Build da imagem do serviço flag-service

docker build -t targeting-service:1.0 .


==============================

### evaluation-service

# desafios

Necessário comentar:

evaluator.go

import (
	//"context"
  "os" // Adicionar

erro:

 > [builder 6/6] RUN CGO_ENABLED=0 GOOS=linux go build -o /evaluation-service .:
7.298 # evaluation-service
7.298 ./evaluator.go:4:2: "context" imported and not used
7.298 ./evaluator.go:106:12: undefined: os
7.298 ./evaluator.go:133:12: undefined: os


## Build da imagem do serviço flag-service

docker build -t evaluation-service:1.0 .

==============================

### analytics-service

# desafios

  File "/app/app.py", line 10, in <module>
    from flask import Flask, jsonify
  File "/opt/venv/lib/python3.11/site-packages/flask/__init__.py", line 5, in <module>
    from .app import Flask as Flask
  File "/opt/venv/lib/python3.11/site-packages/flask/app.py", line 30, in <module>
    from werkzeug.urls import url_quote
ImportError: cannot import name 'url_quote' from 'werkzeug.urls' (/opt/venv/lib/python3.11/site-packages/werkzeug/urls.py)
[2026-06-25 15:25:07 +0000] [7] [INFO] Worker exiting (pid: 7)
[2026-06-25 15:25:07 +0000] [1] [INFO] Shutting down: Master
[2026-06-25 15:25:07 +0000] [1] [INFO] Reason: Worker failed to boot.

Necessário adicionar Werkzeug==2.2.3 no requirements.txt, pois esta falhando para iniciar a aplicação.


Para realizar testes localmente com uma fila sqs e dynamodb, utilizei via docker o elasticmq

foi necessário ajustar algumas linhas no app.py:

try:
    session = boto3.Session(region_name=AWS_REGION)
    dynamodb_kwargs = {"region_name": AWS_REGION}
    sqs_kwargs = {"region_name": AWS_REGION}
    if SQS_ENDPOINT:
        sqs_kwargs["endpoint_url"] = SQS_ENDPOINT
    sqs_client = session.client("sqs", **sqs_kwargs)
    if DYNAMODB_ENDPOINT:
        dynamodb_kwargs["endpoint_url"] = DYNAMODB_ENDPOINT
    dynamodb_client = session.client("dynamodb", **dynamodb_kwargs)
    log.info(f"Clientes Boto3 inicializados na região {AWS_REGION}")
except NoCredentialsError:
    log.critical("Credenciais da AWS não encontradas. Verifique seu ambiente.")
    sys.exit(1)
except Exception as e:
    log.critical(f"Erro ao inicializar o Boto3: {e}")
    sys.exit(1)