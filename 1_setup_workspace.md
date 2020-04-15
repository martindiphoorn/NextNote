# 1 Setup the workspace

Create a workspace folder called nextNote.

In this tutorial we assume that you have installed the following tools:
- Java 8 or higher
- Maven 3
- Node 8 (not higher)
- Angular CLI 8
- Docker
- Docker Compose (Comes with docker)


## 1.1 Backend setup

Go to: https://start.spring.io/

Fill the form in with:

Generate a **Maven Project** with **Java** and Spring Boot **2.2.2**
> Version 2.2.2 is the latest version at the moment of writing this.

**Project MetaData**  
Group: com.example   
Artifact: nextnote

**Dependencies**
- Spring Boot Actuator
- Spring Boot DevTools
- Spring Data JPA
- Spring Security
- Spring Web
- MySQL Client
- Flyway Migration
- Lombok

Open the options part and select Java 11.

Hit generate project and unzip the downloaded file to the workspace folder and call it `backend`.

Next we need an script which wait for the database server before starting docker boot.
Go to the boot folder and create a docker-files folder. 
This folder will contain files only used by docker builds.

Create a file called `starter.sh` and add the following contents:

```bash
#!/bin/bash

set -e

cmd="$@"

# Do an initial wait
sleep 5

# Wait for the server to become available
until mysql -h "${dbHost}" -P "${dbPort}" -u"${dbUsername}" -p"${dbPassword}" -e 'show processlist'; do
  >&2 echo "Database is unavailable - sleeping"
  sleep 5
done

>&2 echo "Database is up - executing command"
exec $cmd
```

Go to the `backend` folder and create a Dockerfile with the following contents:
```yaml
FROM openjdk:11
LABEL author="Martin Diphoorn"

RUN apt-get update && apt-get install --no-install-recommends -y mariadb-client unzip curl && rm -rf /var/lib/apt/lists/*;

VOLUME /tmp

ADD target/*.jar nxt-note.jar
ADD docker-files/starter.sh /opt/starter.sh

RUN chmod +x /opt/starter.sh

HEALTHCHECK --interval=5s --timeout=2s --retries=12 \
  CMD curl --silent --fail http://localhost:8080/api/management/health || exit 1

ENTRYPOINT ["/opt/starter.sh","java","-jar","nxt-note.jar"]
```
This file is used to build a docker container for the boot backend.

## 1.2 Frontend setup

Execute this in your workspace folder in the command line
```bash
ng new nxt-note --style=scss --directory=frontend
```

Go to the new `frontend` folder and create a `docker-files` folder. 
This folder will contain files only used by docker builds. In this case an nginx configuration file.
Let's create it.

Create an `nginx.conf` file in the `docker-files` folder with the following contents:

```text
server {
    listen       80;
    server_name  localhost;

    location /api {
        proxy_pass http://backend:8080/api;
    }

    location / {
        root /usr/share/nginx/html;
        try_files $uri /index.html;
    }
}
```

Next let's create an docker file for the frontend container. Create a file `Dockerfile` with the following contents:

```yaml
FROM nginx:alpine
LABEL author="Martin Diphoorn"

COPY  docker-files/nginx.conf /etc/nginx/conf.d/default.conf
COPY  dist/nxt-note/ /usr/share/nginx/html

EXPOSE 80 443
CMD [ "nginx", "-g", "daemon off;" ]

```
As you can see the nginx.conf will be copied from the docker_files folder into the container.

## 1.3 Docker-compose setup

The last setup part is the glue between the other two containers.
And we will even add another container for our data.
Create a new folder called `docker` in your workspace.
We will create a script here for building everything.

First we start with a `docker-compose.yml` which will contain the configuration of our Docker containers and will glue them together in their own network.
So let's create the `docker-compose.yml` in the docker folder.

```yaml
version: '3.3'
services:
  db:
    image: mariadb:10
    environment:
      - TZ=Europe/Amsterdam
      - MYSQL_DATABASE=notes
      - MYSQL_ROOT_PASSWORD=n0t3s
    ports:
     - "8306:3306"

  backend:
    build:
      context: ../backend
      dockerfile: ./Dockerfile
    environment:
      - TZ=Europe/Amsterdam
      - dbHost=db
      - dbPort=3306
      - dbName=notes
      - dbUsername=root
      - dbPassword=n0t3s
    ports:
      - "8090:8080"
    depends_on:
      - db

  frontend:
    build:
      context: ../frontend
      dockerfile: ./Dockerfile
    environment:
      - TZ=Europe/Amsterdam
    ports:
      - "8080:80"
    depends_on:
      - backend

```

For our convenience here is a script to build the applications and the containers in one command.
I call it `build.sh` and it has the following contents:
```bash
cd ../backend
mvn package -DskipTests

cd ../frontend
npm run build

cd ../docker
docker-compose build
```

Don't forget to give the file executable rights: `chmod +x build.sh`

> Tip: Under windows you can call this build.bat and you can start it without "./" we assume macOs or linux

It goes to `backend` folder and builds the backend without tests.
This is needed as the test will fail in the current state.
Then it will go the `frontend` folder and build the frontend.
And finally it builds the containers with docker-compose.

Let's do a quick run to test if everything is working.
```bash
./build.sh && docker-compose up
```
`build.sh` will build everything as explained above. And docker-compose up will start the containers.

Expect failures as we did not configure anything in spring yet.
However the database should be reachable under port localhost:8306 with username root and password n0t3s in your db client.
And your browse should be able to reach the frontend at http://localhost:8080/
All these things are configured in the docker-compose file.

That's it. We have a working backend in Spring boot and a frontend in Angular.
We also have an MariaDB database called notes.

Just a few commands for docker-compose

```bash docker-compose up ``` Start everything up  
```bash docker-compose up -d ``` Start everything up and run in deamon mode  
```bash docker-compose down ``` Stop and cleanup everything (advised after an build)  
```bash docker-compose start ``` Start existing containers  
```bash docker-compose stop ``` Pause/Stop containers, but remove nothing  
```bash docker-compose logs ``` Show the logs of the containers  
```bash docker-compose logs -f ``` Show the logs of the containers and keep following them
