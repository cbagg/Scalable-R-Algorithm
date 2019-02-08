# Working with R in High Perform Environments

Most people who works with R know that there are two key problems people have productionizing it. 

* Most tech develpers don't know much about R. 
* R is by default single threaded.

Below I have a solution for both problems.

# Plumber, Docker Swarm, and Traefik to Scale an R Algorithm

To make an R application fit into a technology stack, we stand it up as a RESTful service, which means we can do standard GET, POST, etc logic with it. Developers are very familiar with this logic and will be able to consume it easily as second nature.

However, R is single threaded, which means that if 10 users hit the endpoint at the same time, R will serve one user at a time. The last user could be waiting a while depending on the response time. 

To solve this we are going to use docker swarm, which will allow us to stand up multiple copies of the same application. Docker is a lightweight containerization tech that makes your code scalable and portable. Swarm then spins up as many copies of it as you want, even across multiple servers.

Then we use Traefik to balance between the running copies. Production applications could run hundreds of copies of an app, but my example uses 10.

## Plumber

Plumber decorates our code. There is a file in model_server which controls the logic. I have only the simple example running here. If you GET from the webserver at /plot, you get a histogram, if you GET at /echo, it sends back text, and if you POST to /sum with two numbers, it adds them together.

```
# plumber.R

#* Echo back the input
#* @param msg The message to echo
#* @get /echo
function(msg=""){
  list(msg = paste0("The message is: '", msg, "'"))
}

#* Plot a histogram
#* @png
#* @get /plot
function(){
  rand <- rnorm(100)
  hist(rand)
}

#* Return the sum of two numbers
#* @param a The first number to add
#* @param b The second number to add
#* @post /sum
function(a, b){
  as.numeric(a) + as.numeric(b)
}
```

## Docker

In the file model_server/Dockerfile, we define the image we want docker to run. We use the image from trestletech for plumber, we install the broom package, we load in our code, and we run our code via plumber.

```
FROM trestletech/plumber
MAINTAINER Docker User <docker@user.org>

RUN R -e "install.packages('broom')"

COPY plumber.R /opt/plumber.R

ENTRYPOINT ["R", "-e", "pr <- plumber::plumb(commandArgs()[4]); pr$run(host='0.0.0.0', port=8000,swagger=TRUE)"]
CMD ["/opt/plumber.R"]
```

## Docker-Compose

Here was the configuration for Traefik and Plumber. Traefik will watch docker and know automatically which images are running, because it is dynamically connected to the docker.sock via a volume mount.

We use the replicas flag to tel it how many copies to spin up, and tell it to check port 8000. Traefik also runs a dashboard at 8080 to let you see your running applications.

```
version: '3.3'
services:
  model_server:
    image: plumber_server
    networks:
     - traefik-net
    deploy:
      replicas: 10
      labels:
        - "traefik.port=8000"
        - "traefik.backend=echo"
        - "traefik.frontend.rule=Path:/echo"

  traefik:
    image: traefik:latest
    command: --web --docker --docker.swarmmode --docker.watch --logLevel=DEBUG
    ports:
      - 80:80
      - 8080:8080
      - 443:443
    labels:
      - "traefik.enable=false"
    networks:
      - traefik-net
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /dev/null:/traefik.toml
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
networks:
  traefik-net:
    driver: overlay

```

## Docker Swarm

So this is how we get this up and running.

Build our model server. This will will take our plumber.R and create a replicable way to spin it up on demand over and over.

```
docker build -t model_server model_server/
```

Then we start a docker swarm.

```
docker swarm init
```

Then we launch our docker swarm. It will take a few minutes for all 10 images to run.

```
docker stack deploy --compose-file=docker-compose.yml load_balancer
```

Now if you hit localhost/plot over and over, each time it will load balance to a different of your 10 running containers.

What happens if you are flooded with requests?

Well, the below command would spin up 100 versions of your app to respond.

```
docker service scale model_server=100
```

To stop your swarm, type

```
docker stack rm load_balancer
```
