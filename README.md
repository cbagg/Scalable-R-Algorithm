docker build -t model_server model_server/

docker swarm init

docker stack deploy --compose-file=docker-compose.yml load_balancer

