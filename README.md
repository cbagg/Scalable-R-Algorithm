#Working with R in High Perform Environments

Most people who works with R know that there are two key problems people have productionizing it. 

* Most tech develpers don't know much about R. 
* R is by default single threaded.

Below I have a solution for both problems.

#Plumber for Standard REST

To make an R application fit into a technology stack, we stand it up as a RESTful service, which means we can do standard GET, POST, etc logic with it. Developers are very familiar with this logic and will be able to consume it easily as second nature.

However, R is single threaded, which means that if 10 users hit the endpoint at the same time, R will serve one user at a time. The last user could be waiting a while depending on the response time. 

To solve this we are going to use docker swarm, which will allow us to stand up multiple copies of the same application. Then we use Traefik to balance between the running copies. Production applications could run hundreds of copies of an app, but my example uses 10.

##Plumber

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

##Traefik

##Docker

##Docker-Compose

##Docker Swarm


```
docker build -t model_server model_server/
```

```
docker swarm init
```

```
docker stack deploy --compose-file=docker-compose.yml load_balancer
```
