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

# Set build time argument
# Can be set to different values in different images when building that image
# Ex: development and production images
ARG DEFAULT_MONGODB_USERNAME='ROOT'
ARG DEFAULT_MONGODB_PASSWORD='pwd'

# Set environment variable  from the ARG which can then be used in server.js
# ENV variable can be overriden when running a container from the image
# If no value is provided for ENV variable when running the container the ARG value from the image will be used
ENV MONGODB_USERNAME $DEFAULT_MONGODB_USERNAME
ENV MONGODB_PASSWORD $DEFAULT_MONGODB_PASSWORD

EXPOSE $PORT

CMD [ "npm", "start" ]