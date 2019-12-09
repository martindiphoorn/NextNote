# 4 Running outside of docker-compose

Often we want to run angular or spring without docker-compose.
So we can test our worker faster without restarting the whole docker.

First we need to know how we can manage containers individually.

## 4.1 Managing containers
In our case we defined 3 containers in the docker-compose file.

 * db: contains MariaDB
 * backend:  contains Spring Boot
 * frontend: contains Angular
 
 Most Docker Compose commands can be used for one container.
 If we only want to start the db we can use:
 
 ```bash
docker-compose up db
```

This can be done for commands like: start, stop, down, build.


## 4.2 Start Spring Boot from the CLI or an IDE
To support starting Boot from the CLI or an IDE, we first create an profile for development (dev).
Create an `application-dev.properties` in the same folder as application.properties with the following content:

```properties
# Set the properties needed for the database
dbHost=127.0.0.1
dbPort=8306
dbName=notes
dbUsername=root
dbPassword=n0t3s
```

With Docker Compose the above properties are set by environment variables.

Now we can use maven to start our application in development mode.

```properties
SPRING_PROFILES_ACTIVE=dev mvn spring-boot:run
```

In your IDE you probably can set the active Spring profile.

## 4.3 Start Angular 6 locally
In Angular 6 we have something called environments and give us the same behaviour as profiles in Spring Boot.
Let's add an development environment configuration to the angular.json file. 

Angular contains two environments by default:
* environment.ts
* environment.prod.ts

Environment.ts can be seen as the default development profile.
When deploying/building with production profile it will overwrite environment with the environment.prod.ts.

So if we want to change our development setting it can be done in environment.ts.


We want to create an environment property which holds the api endpoint.
So we can change it in in docker or production. Let's add this property

environment.ts
```properties
export const environment = {
  production: false,
  api_endpoint: 'http://localhost:8090'
};
``` 

> Don't forget to change the existing files

We now need to adjust our services (note-api.service.ts and group-api.service.ts).
So it will use the api_endpoint specified in the environment file.

In the `note-api.service.ts` file change the following line
```java
private API_URL = 'http://localhost:8090/notes';
```
into
```java
private API_URL = environment.api_endpoint + '/notes';
```

Do the same for the `group-api.service.ts`, but let it end on '/groups' instead of '/notes'.

After this we can build the frontend with an production profile:

```bash
ng build --configuration=production
```

When we want to run it locally we use:

```properties
npm run start
```

The package.json contains: 

```json
{
  "name": "nxt-note",
  "version": "0.0.0",
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build",
    "test": "ng test",
    "lint": "ng lint",
    "e2e": "ng e2e"
  },
```

`npm run start` will call start in the scripts node and run `ng serve`.


## 4.4 Making it work together

To run our code locally for faster development we need to set the right properties.

When run locally we can't use the hostnames defined in the docker-compose so everything should run throug localhost.
In the docker-compose.yml we defined some port properties.

| container | localhost | docker-network |
| --------  | --------- | -------------- |
| db        | 8306      | 3306           |
| frontend  | 8080      | 80 |
| backend   | 8090      | 8080 |

So if we want to make our local instances work nicely we should change some things.

if the frontend needs to talk to the backend we should change the environment.ts
```json
api_endpoint: 'http://localhost:8090'
```

The backend needs to talk to the database on a different port so we need to change the application-dev.properties

```properties
dbPort=8306
```

This probably are the current values. Changes are that we should make a seperate file for the docker now.
Can you do that?
