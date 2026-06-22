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