minikube start --driver=docker --network=minikube-custom

minikube image load custom-postgres:17
minikube image load custom-redis:7.2

minikube mount /home/vinicius.mendes/Kubernetes/volumes/:/mnt/volumes

psql -h 172.16.2.2 -p 30440 -U flag_service -d flags_db
