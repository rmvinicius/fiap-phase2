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

## Criação de uma database local

# Build do postgres

cd ./flag-service/db/Dockerfile
docker build -t flag-service-postgres:17 .

## Build da imagem do serviço flag-service

docker build -t flag-service:1.0 .