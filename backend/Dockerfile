FROM node

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

# Set build time argument
# Can be set to different values in different images when building that image
# Ex: development and production images
ARG DEFAULT_PORT=80

# Set environment variable which can then be used in server.js
# The default value 80 can be overriden during image creation in command line if needed for the container
ENV PORT $DEFAULT_PORT 

EXPOSE $PORT

CMD [ "npm", "start" ]