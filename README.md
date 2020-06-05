# My tiny news app

Author: Armel Chausse

## Introduction

This tiny news app was made for the Cloud Computing course at CentraleSupélec,
in order to demonstrate my ability to create and deploy Docker images to
DockerHub.
It is composed of two services: a frontend made with React and TypeScript, and a
Python (Flask) backend. It uses the news API (https://newsapi.org/) in order to
fetch the latest news from plenty of sources, and allows the user to filter
these news according to several categories and by country (only France, Germany,
UK and USA are implemented in this early version).

## Prerequisites

In order to run the app, you need a working Docker installation on your
computer, as well as Docker Compose.

## Installation

In order to install the app, clone this repository by entering the following
command in your terminal:

    git clone git@github.com:Lemraus/tiny-news-app.git

then type:

    cd tiny-news-app
    docker-compose up

It will then pull both of the required images from DockerHub
(lemraus/news-app-backend and lemraus/news-app-frontend) and run the app. In
order to see the app, visit http://localhost:3000 from your favorite web
browser.

## Dockerfiles & docker-compose.yml 

### Backend Dockerfile

```dockerfile
FROM python:3.8.3-alpine
RUN mkdir /app
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
ENTRYPOINT ["flask"]
CMD ["run", "--host=0.0.0.0"]
```

The starting point of our image is a Python image (version 3.8.3) running on the
Alpine distribution. We start by creating the /app directory and setting is as
the working directory. Then we copy the pip requirements file (obtained with
`pip freeze > requirements.txt`) in the working directory (`/app`), and install
the requirements listed in it. Once this is done, the content of the current
directory (in our local context) is copied into the container's working
directory. Finally, we run the app by calling `flask run --host=0.0.0.0`, this
value of the host flag meaning that we expose the app to systems that live
outside of our container (our computer, for example). In our source code we
defined the port used to access the app as 5000, so this is the port that will
be used. The `ENTRYPOINT` is the base command that will never change, while the
`CMD` can vary if the person creating a container from this image wants to do
something else than `run`. The `EXPOSE` command is not needed here as our
`docker-compose.yml` file takes care of that part when the services are brought
together.

### Frontend Dockerfile

```dockerfile
FROM node:14.3.0-alpine
RUN apk update && apk add yarn && yarn global add serve && mkdir /app
WORKDIR /app
COPY package.json yarn.lock .
RUN yarn
COPY . .
RUN yarn build
ENTRYPOINT ["serve", "-s", "build", "-l", "3000"]
```

Here we start from a NodeJS image (version 14.3.0) running on the Alpine
distribution. I chose to use Yarn as my package manager for this React app so we
need to install it, it is done by running `apk update && apk add yarn`. I also
need serve to serve my app's static build, so I run `yarn global add serve`.
Then, we create the `/app` directory and we set it as the working directory. The
reason why all these commands are done in the same `RUN` is because these steps
will never change, therefore there is no need to create intermediate images
(like there would be if we used several `RUN` statements). The next step is to
copy the `package.json` and `yarn.lock` files to the working directory (`/app`)
and run `yarn` in order to install the required dependencies. Once this is done,
the content of the local current directory (the source code) is copied into the
container's working directory, then we run `yarn build` to generate a static
build for our application. Finally, we serve the previously created static build
on port 3000. The `EXPOSE` command is not needed here as our `docker-compose.yml`
file takes care of that part when the services are brought together.

### docker-compose.yml

    version: '3'

    services:
      backend:
        image: lemraus/news-app-backend:latest
        container_name: news-app-backend
        expose: 
          - 5000
        ports: 
          - 5000:5000
        environment:
          - FLASK_ENV=production
          - FLASK_APP=app.py
          - FLASK_DEBUG=0

      frontend:
        image: lemraus/news-app-frontend:latest
        container_name: news-app-frontend
        expose: 
          - 3000
        ports: 
          - 3000:3000

Two services are created: `backend` and `frontend`. We indicate the `image` from
which their containers are created, and their respective `container_name`. We
then `expose` the adequate port for each container (5000 for `backend`, 3000 for
`frontend`), and map our local machine's ports to the containers' ports. An
extra step is added for the `backend` service: we add `environment` variables
to specify what behaviour Flask should have (production environment, no debug
and an entrypoint being the `app.py` file of our source code).
