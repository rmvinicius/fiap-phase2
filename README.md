# Construção do ambiente no lab aws

1- Criado a estrutura de rede, dividindo em subnets para eks, postgres, redis e demais serviços
2- Criar security groups para externo e interno, no sg externo, permiti acesso a subnet do ec2, no qual foi provisionada uma maquina para administrar o cluster
3- Criar as route table, separando externo e interno também
4- Criado IGW para tráfego de entrada e saída
5- Não foi criado NAT Gateway, devido ao custo de alocar IP publico

# Provisionamento dos recursos

1- Provisionado 3 instâncias RDS
2- Provisionado um ECR
3- Provisionado um Redis
4- Provisionado um EKS
  Desafio: Foi necessário adicionar nas subnets do eks, auto public ipv4
5- Provisionado a tabela do dynamodb

# Configuração dos recursos

1- Na VM que será utilizada para administrar o cluster, foi obtido as credenciais da aws pelo lab, acessando o arquivo ~/.aws/credentials
2- Configurado na VM as credenciais através do comando aws configure
3- Configurado a conexão no cluster através do comando: aws eks update-kubeconfig --region us-east-1 --name eks-dev

## Configuração do postgres

## auth_db

export RDSHOST01="database-dev-01.cnyn2rd2lout.us-east-1.rds.amazonaws.com" 
psql "host=$RDSHOST01 port=5432 dbname=postgres user=postgres"

CREATE DATABASE auth_db;
\c auth_db;
CREATE USER auth_service WITH PASSWORD 'auth_service@2026';
GRANT ALL PRIVILEGES ON DATABASE auth_db TO auth_service;
ALTER DATABASE auth_db OWNER TO auth_service;
ALTER SCHEMA public OWNER TO auth_service;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO auth_service;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO auth_service;

## flags_db

export RDSHOST02="database-dev-02.cnyn2rd2lout.us-east-1.rds.amazonaws.com" 
psql "host=$RDSHOST02 port=5432 dbname=postgres user=postgres"

CREATE DATABASE flags_db;
\c flags_db;
CREATE USER flag_service WITH PASSWORD 'flag_service2026';
GRANT ALL PRIVILEGES ON DATABASE flags_db TO flag_service;
ALTER DATABASE flags_db OWNER TO flag_service;
ALTER SCHEMA public OWNER TO flag_service;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO flag_service;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO flag_service;

## targeting_db

export RDSHOST03="database-dev-03.cnyn2rd2lout.us-east-1.rds.amazonaws.com"
psql "host=$RDSHOST03 port=5432 dbname=postgres user=postgres"

CREATE DATABASE targeting_db;
\c targeting_db;
CREATE USER targeting_service WITH PASSWORD 'targeting_service2026';
GRANT ALL PRIVILEGES ON DATABASE targeting_db TO targeting_service;
ALTER DATABASE targeting_db OWNER TO targeting_service;
ALTER SCHEMA public OWNER TO targeting_service;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO targeting_service;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO targeting_service;


# Configuração autenticação no ECR pela maquina linux para fazer build das imagens

aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 076892642827.dkr.ecr.us-east-1.amazonaws.com

docker build -t toggle-master/auth-service:1.0 .
docker tag toggle-master/auth-service:1.0 076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/auth-service:1.0
docker push 076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/auth-service:1.0

docker build -t toggle-master/targeting-service:1.0 .
docker tag toggle-master/targeting-service:1.0 076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/targeting-service:1.0
docker push 076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/targeting-service:1.0

docker build -t toggle-master/flag-service:1.0 .
docker tag toggle-master/flag-service:1.0 076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/flag-service:1.0
docker push 076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/flag-service:1.0

docker build -t toggle-master/analytics-service:1.0 .
docker tag toggle-master/analytics-service:1.0 076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/analytics-service:1.0
docker push 076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/analytics-service:1.0

docker build -t toggle-master/evaluation-service:1.0 .
docker tag toggle-master/evaluation-service:1.0 076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/evaluation-service:1.0
docker push 076892642827.dkr.ecr.us-east-1.amazonaws.com/toggle-master/evaluation-service:1.0

# Aplicação dos scripts sql nas bases

psql "host=$RDSHOST01 port=5432 dbname=auth_db user=auth_service" -f init.sql
psql "host=$RDSHOST02 port=5432 dbname=flags_db user=flag_service" -f init.sql
psql "host=$RDSHOST03 port=5432 dbname=targeting_db user=targeting_service" -f init.sql

# Configurar ECR no EKS
aws eks update-kubeconfig --region us-east-1 --name eks-dev-01
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 076892642827.dkr.ecr.us-east-1.amazonaws.com
kubectl create secret docker-registry ecr-secret --docker-server=076892642827.dkr.ecr.us-east-1.amazonaws.com --docker-username=AWS --docker-password=$(aws ecr get-login-password --region us-east-1)

# Aplicar deployments do cluster
kubectl apply -f eks/deployment.yaml
kubectl apply -f eks/service.yaml

# Configurar redis

Criado redis do tipo Redis OSS
Necessário criar um usuário para acesso
toggle-master:toggle-master123

## Configurar credenciais no Kubernetes
Connection string:

echo -n "rediss://redis-dev-01-chlpid.serverless.use1.cache.amazonaws.com:6379" | base64

