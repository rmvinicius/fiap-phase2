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

