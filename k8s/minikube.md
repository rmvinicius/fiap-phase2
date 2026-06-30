minikube image load custom-postgres:17

minikube mount /home/vinicius.mendes/Kubernetes/volumes/:/mnt/volumes


 minikube start --driver=docker --network=minikube-custom

 psql -h 172.16.2.2 -p 30440 -U flag_service -d flags_db
