# Step by step Notes

## Getting the app working

* I can run both the backend and front end locally
* So first gonna dockerize mongodb using the official mongo image
* Test the app locally

### Dockerizing the mongodb database

* Image used - https://hub.docker.com/_/mongo
   - `docker run --name mongoctr --rm -d -p 27017:27017 mongo:latest`
   - Documentation says "the image listens on the standard MongoDB port, 27017"
   - Not using a network as the other backend and front end apps are not dockerized
* Backend
   - After dockerizing the mongodb ran the backend by running command `node app.js`
   - Got the output `CONNECTED TO MONGODB` confiming that the backend worked locally with dockerized mongo db

### Dockerizing the backend

* Initial docker file created
* Build a new image for the backend using
   - `docker build -t node_backend_compose:initial .`

### Running the backend and mongodb in docker

* Now we need to run both the backend and mongodb in docker
* This means we need to create a network
   - `docker network create network-compose`
* Now we need to run the mongo container using the official image inside the network
   - We don't need to expose the mongo port as within the mongodb is used within the same network by the backend app
   - Thus mongodb is only used internally by the backend app
   - Prune or remove the previous mongo container
   - `docker run --name mongoctr --rm -d --network network-compose mongo:latest`

### Backend changes to work in a docker network

* Now we need to change the connection string of the backend
   - The backend app connects to mongodb using the below connection string
   - `mongodb://localhost:27017/course-goals`
   - Instead of localhost now we need to use the container name for mongodb
   - Docker resolves using the container name for containers within the same network
   - `mongodb://mongoctr:27017/course-goals`
* We need to rebuild the image to reflect the changes
   - `docker build -t node_backend_compose:initial .`
* We need to run the backend within the same network
   - we do not need to expose the port as we are only planning to use this from the frontend
   - `docker run --name compose-backend --rm -d --network network-compose node_backend_compose:initial`
   - Looking at the logs using `docker logs compose-backend` showed `CONNECTED TO MONGODB`

### Dockerizing the frond end app

* Now we need to dockerize the front end app
   - It's based on the react starter kit so running on npm and express
   - Should run in the same network
   - Should expose the port 3000 which is the default for react startr kit express settings
   - The front end will access the backend but within the same network so thats why we didn't expose any ports in the backend docker container
   - Change the api urls to reflect the image
      - Change the api code in the front end to reference the container by name instead of localhost
      - Ex: `http://localhost/goals/` needs to be `http://compose-backend/goals/`
   - Build the image
      - `docker build -t node_front_compose:initial .`
   - Run a container based on that image
      - `docker run --name compose-front --rm -d --network network-compose -p 3000:3000 node_front_compose:initial`

### Why doesn't the frontend work after dockerizing

* The frontned app uses node and express to run the app
   - I used a docker network to start it and changed all the backend calling get, post api calls to reflect a change in api address by using teh network like `http://localhost/goals/` needs to be `http://compose-backend/goals/`
   - The froend end is a react app that runs on the browser
   - That means its not actually running in the container and when the javascript tries to call the backend api its already running outside of the container and in the browser
   - Only the node server thats serving the html and javascript are runnin in the container
   - Thus javascript doesn't understand the backend api url that is using the network name
   - So the frontend actually doesn't use the network at all so doesn't even need to be on the same network
* In order to fix this issue
   - The backend api needs to espose the port and make the backend accessible from the localhost machine
      - Lets run the backend again using the same image (image already have the port export. We just didn't expose it to localhost when running the container last time)
      - `docker run --name compose-backend --rm -d --network network-compose -p 8080:80 node_backend_compose:initial`
   - Frontend needs to call the backend api using the exposed port 
      - Change all the api code to look like `http://localhost:8080/goals/`
   - Build the image
      - `docker build -t node_front_compose:initial .`
   - Run a container based on that image. We don't need it to be in the network though we do it to contain everything together
      - `docker run --name compose-front --rm -d --network network-compose -p 3000:3000 node_front_compose:initial`

## MongoDB - Storage and Security

* Now that we have the application working we can focus on improvements. 

### MongoDB presistance on container destruction

* Mongodb is a document database
* The data is currently stored inside of the running container based on the image
* We are using the official image [docker-hub-mongo](https://hub.docker.com/_/mongo)
* We have to either keep using the same container in order to presist the data between contaienr restarts
* If we use `--rm` flag that means data is lost once th econtainer exists
* The official documentation specifies that the data is stored in the container at path `/data/db`
* So that means we can use a volume to presist the data 
* Official documentation also specified that mac os file system is not compatible with the memory mapped filed used by mongodb
* Meaning we cannot use a bind mount cause in a bind mount we are mapping a host folder to a container
* Instead we can get docker to manage the volue by using a named volume. (Well an anonymous volme doesn't achieve the requirement of reusability of data across containers)
* So lets stop the running container and start it with a named volume
   - `docker stop mongoctr`
   - `docker run --name mongoctr --rm -d --network network-compose -v db-volume:/data/db mongo:latest`

### Securing the MondoDB using authentication

* Currently we do not use any auth for the mongodb db
* The official image supports setting a username and a passsword when initializing the database
* Ideally we should not store the plaintext username and password 
   - Docker has a concept of docker secrets in swarm mode
   - Or there are other services we can use
   - For now lets go with plaintext to test
* Mongo does support role based authentication and have docs provided on the official image. However lets try something simple first
* Mongodb official image supports setting the username and password using environment variables at initalization
   - Stop the running container
   - `docker stop mongoctr`
   - IMPORTANT: Delete the datavolume as it has the old database with the default password
   - `docker volume remove db-volume`
   - Initialize a new container and override the default username password by overriding the environment variables
   - `docker run --name mongoctr --rm -d --network network-compose -v db-volume:/data/db -e MONGO_INITDB_ROOT_USERNAME=devuser -e MONGO_INITDB_ROOT_PASSWORD=pwd mongo:latest`
   - Now change the connection string of the backend to send the user and password in plain text query string for the time being
   - `mongodb://devuser:pwd@mongoctr:27017/course-goals?authSource=admin`
   - Stop the container, rebuild the image and run a new container for the backend
   - `docker stop compose-backend`
   - `docker build -t node_backend_compose:initial .`
   - `docker run --name compose-backend --rm -d --network network-compose -p 8080:80 node_backend_compose:initial`

## Fine tuning for development and prod

### Backend app set port using arguments and environment variables

* Currently the port is hard coded
* Use an ENV variable and a ARG combo
   - ARG is used to set it and this can be overriden in build time (ex: debug and production builds can have different values thus we can override and build different images for debug and prod)
   - ENV variable is set using the argument inside the docker file. ENV variable can be used by the server.js file (inside code) thus its important to use that. ENV variables can also be overriden during runtime so we can override the images ARG when starting a container using that image if needed.
* Then
   - Stop the container `docker stop compose-backend`
   - Rebuild the image and thistime specify the ARG (to test give this as 8080 for the image so when running we can override to 81 in container)
      - `docker build --build-arg DEFAULT_PORT=8080 -t node_backend_compose:initial .`
   - Run a container with that image
      - `docker run --name compose-backend -e PORT=81 --rm -d --network network-compose -p 8080:81 node_backend_compose:initial`

### Backend app volumes and auto refresh with code changes