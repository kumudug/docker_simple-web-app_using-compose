# Step by step Notes

## Getting the app working

* I can run both the backend and front end locally
* So first gonna dockerize mongodb using the official mongo image
* Test the app locally

### Dockerizing the mongodb database

* Image used - https://hub.docker.com/_/mongo
   - `docker run --name mongoctr -d mongo:latest -p 27017:27017`
   - Documentation says "the image listens on the standard MongoDB port, 27017"
   - Not using a network as the other backend and front end apps are not dockerized