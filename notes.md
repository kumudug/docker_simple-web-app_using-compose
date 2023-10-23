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