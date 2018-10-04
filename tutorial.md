# 1 Setup the workspace

Create a workspace folder called nextNote.

In this tutorial we assume that you have installed the following tools:
- Java 8 or higher
- Maven 3
- Node 8 (not higher)
- Angular CLI 6
- Docker
- Docker Compose (Comes with docker)


## 1.1 Backend setup

Go to: https://start.spring.io/

Fill the form in with:

Generate a **Maven Project** with **Java** and Spring Boot **2.0.4**
> Version 2.0.4 is the latest version at the moment of writing this.

**Project MetaData**  
Group: com.example   
Artifact: nextnote

**Dependencies**
- JPA
- Web
- Security
- DevTools
- Lombok
- Flyway

Hit generate project and unzip the downloaded file to the workspace folder and call it boot.

Next we need an script which wait for the database server before starting docker boot.
Go to the boot folder and create a docker-files folder. 
This folder will contain files only used by docker builds.

Create a file called starter.sh and add the following contents:

```bash
#!/bin/bash

set -e

cmd="$@"

# Do an initial wait
sleep 5

# Wait for the server to become available
until mysql -h "${dbHost}" -P "${dbPort}" -u"${dbUsername}" -p"${dbPassword}" -e 'show processlist'; do
  >&2 echo "MariaDB is unavailable - sleeping"
  sleep 2
done

>&2 echo "MariaDB is up - executing command"
exec $cmd
```

Go to the boot folder and create a Dockerfile with the following contents:
```yaml
FROM openjdk:8
LABEL author="Martin Diphoorn"

RUN apt-get update && apt-get install --no-install-recommends -y mariadb-client unzip && rm -rf /var/lib/apt/lists/*;

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
ng new nxt-note --style=scss
```

Go to the new nxt-note folder and create a docker-files folder. 
This folder will contain files only used by docker builds. In this case an nginx configuration file.
Let's create it.

Create an docker-files/nginx.conf file with the following contents:

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

Next let's create an docker file for the frontend container. Create a Dockerfile with the following contents:

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
Create a new folder called ```docker``` in your workspace.
We will create a script here for building everything.

First we start with a docker-compose.yml which will contain the configuration of our Docker containers and will glue them together in their own network.
So let's create the docker-compose.yml in the docker folder.

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
      context: ../boot
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
      context: ../nxt-note
      dockerfile: ./Dockerfile
    environment:
      - TZ=Europe/Amsterdam
    ports:
      - "8080:80"
    depends_on:
      - backend

```

For our convenience here is a script to build the applications and the containers in one command.
I call it build.sh and it has the following contents:
```bash
cd ../boot
mvn package -DskipTests

cd ../nxt-note
npm run build

cd ../docker
docker-compose build
```

Don't forget to give the file executable rights: `chmod +x build.sh`

> Tip: Under windows you can call this build.bak and you can start it without "./" we assume macos or linux

It goes to boot folder and builds the backend without tests.
This is needed as the test will fail in the current state.
Then it will go the nxt-note folder and build the frontend.
And finally it builds the containers with docker-compose.

Let's do a quick run to test if everything is working.
```bash
./build.sh && docker-compose up
```
build.sh will build everything as explained above. And docker-compose up will start the containers.

Expect failures as we did not configure anything in spring yet.
However the database should be reachable under port localhost:8306 with username root and password n0t3s in your db client.
And your browse should be able to reach the frontend at http://localhost:8080/
All these things are configured in the docker-compose file.

That's it. We have a working backend in Spring boot and a frontend in Angular 6.
We also have an MariaDB database called notes.

Just a few commands for docker-compose

```bash docker-compose up ``` Start everything up  
```bash docker-compose up -d ``` Start everything up and run in deamon mode  
```bash docker-compose down ``` Stop and cleanup everything (advised after an build)  
```bash docker-compose start ``` Start existing containers  
```bash docker-compose stop ``` Pause/Stop containers, but remove nothing  
```bash docker-compose logs ``` Show the logs of the containers  
```bash docker-compose logs -f ``` Show the logs of the containers and keep following them

# 2 Backend boot
So now we have everything setup we can start working on the backend. The backend will be created with spring boot.
Spring boot comes with a lot of helper libraries to make our live easier.
But before we can do that we first need to setup the database.

We will do that and more in the application.properties 

## 2.1 Application properties
application.properties is located in the resource folder of the project (boot/src/main/resources/application.properties) and probably is empty right now.
In Spring boot it is possible to have a properties file or an yaml file. We will be using the properties.

Also it is possible to have different properties for different environments. We will be using the default one.

We need to make Spring boot aware of our datasource, so let's add these lines:

```properties
# Datasource configuration
spring.datasource.url=jdbc:mysql://${dbHost}:${dbPort}/${dbName}
spring.datasource.username=${dbUsername}
spring.datasource.password=${dbPassword}
spring.datasource.driverClassName=com.mysql.jdbc.Driver

# Validate entities against the database during startup
spring.jpa.hibernate.ddl-auto=validate

# Make sure hibernate don't map groupId to group_id
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
```

This is all spring needs to know. However it will not start, because mysql is not yet added to the project.
Let's add the mysql dependency to our pom.xml. Add the dependency below to your pom.xml



```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

That's it, we have made an db connection. If you want to test it,
 you will need to shutdown your current docker-compose by pressing ctrl + c.
Next you can run the same command as before:

```bash
./build.sh && docker-compose up
``` 

When it is starting you will notice that it will still fail on flyway configuration.
But that is the next chapter. So let's continue and configure flyway.

## 2.2 Flyway and migrations

Actually we don't need to configure flyway, it is configured by default. Remember we added flyway to the initializer?
That's it. In the pom.xml file you will see that it is configured as starter. Spring does the rest.

But what is the error we see? Well let's check te error:
```composer log
backend_1   | Caused by: java.lang.IllegalStateException: Cannot find migrations location in: [classpath:db/migration] (please add migrations or check your Flyway configuration)
backend_1   | 	at org.springframework.util.Assert.state(Assert.java:94) ~[spring-core-5.0.8.RELEASE.jar!/:5.0.8.RELEASE]
```

The important part of this error is ```please add migrations or check your Flyway configuration``` let's do that.

By default flyway checks resources/db/migration. 
You already have resources that is where your application.properties is.
Create a folder ```db``` in the resources folder. Create a ```migration``` folder in the just created db folder.

Now we need to add an sql file in the migration folder.
Create a file called ```V00001__initial.sql``` in the migration folder.
Be aware that after the Version number there are two underscores, if you miss that it will fail.

```sql
CREATE TABLE IF NOT EXISTS `note` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(50) NOT NULL,
  PRIMARY KEY (`id`))
  ENGINE = InnoDB
  DEFAULT CHARACTER SET = utf8;
```

Now let's build an start. Now it should work and start without errors!

If you have an database viewer or client you can now check what has been done in your database.

You will notice that there are two tables added to your database.
One is our note table from the create script.
The other one is called flyway_schema_history and will be used by flyway to keep track of the migrations.

So know we have an note table, but maybe we want to be able to group notes.
Time for our second migration file.

Create a file called ```V00002__group.sql``` in the migration folder.

```sql
CREATE TABLE `notegroup` (
  id   BIGINT NOT NULL AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL,
  CONSTRAINT `PRIMARY` PRIMARY KEY (id)
)
  ENGINE = InnoDB
  DEFAULT CHARSET = utf8
  COLLATE = utf8_general_ci;

ALTER TABLE `note`
  ADD groupId BIGINT;

ALTER TABLE `note`
  ADD CONSTRAINT FK_NoteGroup
  FOREIGN KEY (groupId) REFERENCES notegroup(id);
```

Now we have a backend running and a database we can continue to the entities.

## 2.3 Entities and lombok

Entities are the mapping between your database tabel and the java code.
Most of the time it contains the same fields and relations as in the database.
We will use lombok here so we can forget about the getters and the setters.

For our structure we try to use functional folders. So group and note will receive seperate folders.
Check the package names of the files for the right folder.

Group.java
```java
package com.example.nextnote.group;

import javax.persistence.*;

import lombok.Data;

@Data
@Entity
@Table(name = "notegroup")
public class Group {
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	private String name;
}

```

Note.java
```java
package com.example.nextnote.note;

import javax.persistence.*;
import javax.validation.constraints.NotNull;

import lombok.Data;

import com.example.nextnote.group.Group;

@Data
@Entity
@Table(name = "note")
public class Note {
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "groupId")
	private Group group;

	private String name;
}
```

So as you can see the entities are staying cleaner because of the @Data lombok will create public getId and setId methods.
With the entities in place, we should build and restart docker-compose again. 
In the application.properties we added the following property:

```properties
# Validate entities against the database during startup
spring.jpa.hibernate.ddl-auto=validate
``` 
So during the next start our database will be validated against above entities. Hopefully we did everything right!
Everything alright? Let's move on to the repositories.

## 2.4 Spring data Repositories

When you want to retrieve data you need create sql queries for the retrieval. 
And create connections and other boring hibernate/jpa stuff.
Spring Data will help you with this and you just need to create an interface.
Spring Data will create the implementation. Spring calls these interfaces repositories.

Create a new file called NoteRepository.java in the note folder.

```java
package com.example.nextnote.note;

import org.springframework.data.repository.CrudRepository;

public interface NoteRepository extends CrudRepository<Note, Long> {
}
```

By creating this we have the following methods available:
>* count()
>* delete(Note note)
>* deleteAll()
>* deleteAll(Iterable<? extends Note> notes)
>* deleteById(Long id)
>* existsById(Long id)
>* findAll()
>* findAllById(Iterable<Long> ids)
>* findById(Long id)
>* save(Note note)
>* saveAll(Iterable<Note> notes)  

*This list is copied from the: [Spring documentation](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html)*

These are the most common used methods in an application, but if we need more we can create that.
But for now this is enough.

I expect that you can do this on your own for the group part.
So go ahead and make the GroupRepository interface in the group folder. 

## 2.5 Rest security
At this stage we are not going to implement a fully featured security model.
But as we added the security starter from Spring we need to allow everything for easy testing.

Create a folder config and in that folder a file called WebSecurityConfig.java with the following contents:

```java
package com.example.nextnote.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.config.http.SessionCreationPolicy;

@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity http) throws Exception {

		http
                .csrf().disable() // Disable CSRF
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS) // Create stateless sessions
                .and()
                .authorizeRequests()
                .anyRequest().permitAll(); // Allow everything
	}
}

```

As mentioned in the code it permits everything, do not use this in production/live situations. 
This is for testing only.

Because we are going to use rest services we will encounter CORS exceptions.
To prevent this we need to prepare Spring Boot for it.

So we add another config class called WebConfig.java with the following contents:

```java
package com.example.nextnote.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

	@Override
	public void addCorsMappings(CorsRegistry registry) {
		registry
				.addMapping("/**")
				.allowedOrigins("*")
				.allowedMethods("GET", "POST", "PUT", "DELETE", "HEAD")
				.exposedHeaders("Content-Disposition", "Content-Type");
	}
}
```

## 2.6 Rest controllers
Now we can display our data, we should make it available to the frontend. This will be done by restfull service. 
In Spring these are controllers of data. And for rest we will use the annotation @RestController for it. 

Let's create NoteController.java in the folder note and replace it with the following contents:

```java
package com.example.nextnote.note;

import java.util.Optional;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
public class NoteController {

	private final NoteRepository noteRepository;

	/**
	 * Spring will automatically inject the noteRepository
	 *
	 * @param noteRepository
	 */
	@Autowired
	public NoteController(NoteRepository noteRepository) {
		this.noteRepository = noteRepository;
	}

	/**
	 * Returns all our notes from the database
	 *
	 * @return
	 */
	@RequestMapping(value = "/notes", method = RequestMethod.GET)
	public Iterable<Note> all() {
		return this.noteRepository.findAll();
	}

	/**
	 * Return the note with the specified id or null if is not available
	 *
	 * @param id
	 * @return
	 */
	@RequestMapping(value = "/notes/{id}", method = RequestMethod.GET)
	public Note one(@PathVariable("id") Long id) {
		return this.noteRepository.findById(id).orElse(null);
	}

	/**
	 * Create a new note
	 *
	 * @param note
	 * @return
	 */
	@RequestMapping(value = "/notes", method = RequestMethod.POST)
	public Note create(@RequestBody Note note) {
		note.setId(null); // There should not be an id when creating a new record
		this.noteRepository.save(note);
		return note;
	}

	/**
	 * Update the note with the given id
	 *
	 * @param id
	 * @param note
	 * @return
	 */
	@RequestMapping(value = "/notes/{id}", method = RequestMethod.PUT)
	public Note update(@PathVariable("id") Long id, @RequestBody Note note) {
		// Retrieve the note by the id
		Optional<Note> optionalNote = this.noteRepository.findById(id);
		if (optionalNote.isPresent()) {
			Note dbNote = optionalNote.get();
			dbNote.setName(note.getName());
			this.noteRepository.save(dbNote);
			return dbNote;
		}

		return null;
	}

}
``` 
> **Note 1**: we are working here straight on the database entities. 
> Normally I would prefer to use data transfer objects (DTO) for this.
> For simplicity reasons I left it out for now.
> This will be added with MapStruct in Chapter 5.

> **Note 2**: There is no security or proper fault handling. There is room for improvement. 
> But again, for simplicity reseasons I left it out

Now go ahead and do the same for groups.

After this you can build and restart the docker containers.

> use ```./build.sh && docker-compose up``` in the docker folder to start it 

Now you can test your restfull services with postman, soapui or an other tool.
A quick test could be to run it in your browser with the following url:
> http://localhost:8090/notes

The database is still empty so it would return an empty array.

Time to continue with the angular part.

# 3 Frontend angular 
During the setup part we have run: 

```bash
ng new nxt-note --style=scss
```

This created a new project called nxt-note and we are now going to extend it. 
We have add --style=scss to tell angular that we don't want to use old style sheets.

If you want to know more about scss style sheets check out the sass lang page.

https://sass-lang.com/guide

Watch out that you check the SCSS examples and not the SASS ones.
SCSS is compatible with normal css, but allows much more like nesting.


## 3.1 Components all over the place
Angular 6 uses components for everything. Navigation is also done by changing components.
Even the start of the application is a component. It is called app.component.ts 

So let's create a new component for our note list. Which gives us an overview of notes.
We use the angular cli to create a new component. Let's create our first component with:

```bash
ng generate component NoteList
```

this will create a functional folder called note-list with four files:

>* note-list.component.scss
>* note-list.component.html
>* note-list.component.spec.ts
>* note-list.component.ts

Scss is the sass stylesheet, html is the layout ts is the typescript file for the component.
And spec.ts files are used for unit testing and will not be distrubated.   

Feels good? Let's create our second component:

```bash
ng generate component Note
```

## 3.2 Routing for navigation

We also need to make it possible to reach the components. To do this we need to make some routing.
The basics can be generated with angular cli, let's run the following command:

```bash
ng generate module app-routing --flat --module=app
```
> --flat puts the file in src/app instead of its own folder.
> --module=app tells the CLI to register it in the imports array of the AppModule.

This will generate an app-routing.module.ts which is loaded from the app.module.ts.
So now we have a new module named AppRouting and it is loading by the app.module.ts.

Now let's open the app-routing.module.ts and make it a routing module, by adding some configuration and add some routes to our current note and note list components.

```typescript
import {NgModule} from '@angular/core';
import {RouterModule, Routes} from "@angular/router";
import {NoteListComponent} from "./note-list/note-list.component";
import {NoteComponent} from "./note/note.component";

const routes: Routes = [
  { path: '', redirectTo: '/notes', pathMatch: 'full' },
  { path: 'notes', component: NoteListComponent },
  { path: 'note/:id', component: NoteComponent }
];


@NgModule({
  imports: [ RouterModule.forRoot(routes) ],
  exports: [ RouterModule ]
})
export class AppRoutingModule { }
```

So let's explain this a bit. In the constant routes we define the paths to our components.
You see that the path to note contains an placeholder :id which can be used to open the note by it's id in the detail page.
The last line is the default redirect if nothing is given. The option pathMatch tells the router that is should be an exact match with nothing else behind it.

Next we create an import of the RouterModule by using forRoot with our routes constant.
forRoot is often used by modules that contain parts that should only live once during the application (singleton).
In our case we only want one RouterModule.

We still need to tell the RouterModule where to output the components we just navigated to.

Open app.component.html and change the contents to:

```html
<h1>{{title}}</h1>
<router-outlet></router-outlet>
```

The contents of ```{{title}}``` can be found in app.component.ts, open it and change the title to 'Next Notes'.

Now we can now test our routing by starting up docker compose and changing the url by hand.

> use ```./build.sh && docker-compose up``` in the docker folder to start it

Now open up a browser and type in: http://localhost:8080/   
If you have watch the url you should have noticed that it has changed to /notes.
Let's change it to /note/1 and we should see a page with: note works!

We just finished creating the base routing for our application.

Now we will add a link from the list to the note.
Let's open note-list.component.html and replace the contents with:

```html
<h2>Notes overview</h2>
<p>
  this is a test link to <a routerLink="/note/1">Note 1</a><br/>
  this is a test link to <a routerLink="/note/2">Note 2</a>
</p>
```

When we are in the note page we want to display the note id and also a link back to the notes.
Open note.component.html and change the content into:

```html
<h2>Note details</h2>
<a routerLink="/notes">Back to the overview</a>
<p>
  this will become the details page of our note. You reached this note with id: {{id}}
</p>
```

Next we need to open the note.component.ts and make the id available for the html.
The comments will try to explain the code, you can skip that.

```typescript
import { Component, OnInit } from '@angular/core';
import {ActivatedRoute} from "@angular/router";

@Component({
  selector: 'app-note',
  templateUrl: './note.component.html',
  styleUrls: ['./note.component.scss']
})
export class NoteComponent implements OnInit {
  // the html will retrieve this variable for displaying
  id: number;

  // variables in the constructor will be injected by angular
  // By making it private we can use it in the class with this.route
  // Without private it is only available in the constructor.
  constructor(private route: ActivatedRoute) { }

  // ngOnInit is a lifecycle hook and will be called when all injections/inputs have been handled.
  ngOnInit() {
    this.getNote();
  }

  // This is now just a way to set the id. But we will change this later to retrieve the note.
  getNote(): void {
    this.id = +this.route.snapshot.paramMap.get('id');
  }
}
``` 

Now rebuild and start your containers. 
> If you don't know how you can read back

## 3.3 Services for data retrieval
Before we can continue with creating or updating notes we first need to create the communication with the backend.
We will do this with injectable services. But first we need to add the HttpClient to our project.

Open app.module.ts and add the following lines:

```typescript
...
import {HttpClientModule} from "@angular/common/http";
...

@NgModule({
  ...

  imports: [
    BrowserModule,
    HttpClientModule,
    AppRoutingModule,
  ],
  
  ...
```

> The dots (...) means that we skipped lines, just add the missing lines to your file.

So now we can make http connections for data retrieval. Let's create our first service.
Again we use the cli to create the base. We put in the note/shared folder so we know it will be shared through te application.

```bash
ng generate service note/shared/NoteApi
```

Open the new note-api.service.ts file and change it to the following code.

```typescript
import {Injectable} from '@angular/core';
import {HttpClient} from "@angular/common/http";

@Injectable({
  providedIn: 'root'
})
export class NoteApiService {

  private API_URL = 'http://localhost:8080/notes';

  constructor(private  httpClient: HttpClient) {
  }

  public all() {
    return this.httpClient.get(this.API_URL);
  }

  public get(id: number) {
    return this.httpClient.get(this.API_URL + '/' + id);
  }
}
``` 

We now have a simple api which can be used to retrieve all or one note from the backend.
However there is no type safety. We don't know what will come back. Let's add some type safety.

First we need to create a note object in typescript. Create a note.model.ts in the note/shared folder with the following contents:

```typescript
export class Note {
  id: number;
  name: string;
}
```

Next we update the NoteApiService with the typesafety in place:

```typescript
import {Injectable} from '@angular/core';
import {HttpClient} from "@angular/common/http";
import {Note} from "./note.model";
import {Observable} from "rxjs";

@Injectable({
  providedIn: 'root'
})
export class NoteApiService {

  private API_URL = 'http://localhost:8080/notes';

  constructor(private  httpClient: HttpClient) {
  }

  public all(): Observable<Note[]> {
    return this.httpClient.get<Note[]>(this.API_URL);
  }

  public get(id: number): Observable<Note> {
    return this.httpClient.get<Note>(this.API_URL + '/' + id);
  }
}
``` 

Now we need to change the note-list and note components to retrieve the data and display it.

note-list.component.ts

```typescript
import { Component, OnInit } from '@angular/core';
import {NoteApiService} from "../note/shared/note-api.service";
import {Note} from "../note/shared/note.model";

@Component({
  selector: 'app-note-list',
  templateUrl: './note-list.component.html',
  styleUrls: ['./note-list.component.scss']
})
export class NoteListComponent implements OnInit {
  notes: Note[];

  constructor(private noteApiService: NoteApiService) { }

  ngOnInit() {
    this.noteApiService.all().subscribe(value => {
      this.notes = value;
    })
  }
}
```

note-list.component.html
```html
<h2>Notes overview</h2>
<ul>
  <li *ngFor="let note of notes" routerLink="/note/{{note.id}}">{{note.name}}</li>
</ul>

```

In angular there are two ways to handle forms. The first is the FormsModule which is similar to AngularJS with ngModel.
The other is ReactiveFormsModule which is more control based.
As most people know the FormsModule style from AngularJS we will use the ReactiveFormsModule here.

So we first need to import the ReactiveFormsModule in the app.module.ts:

app.module.ts
```typescript
...
import {ReactiveFormsModule} from "@angular/forms";
...

@NgModule({
  ...
  imports: [
    BrowserModule,
    HttpClientModule,
    ReactiveFormsModule,
    AppRoutingModule
  ]
  ...

``` 

note.component.ts
```typescript
import {Component, OnInit} from '@angular/core';
import {ActivatedRoute} from "@angular/router";
import {NoteApiService} from "./shared/note-api.service";
import {FormBuilder, FormGroup} from "@angular/forms";
import {Note} from "./shared/note.model";

@Component({
  selector: 'app-note',
  templateUrl: './note.component.html',
  styleUrls: ['./note.component.scss']
})
export class NoteComponent implements OnInit {
  editForm: FormGroup;

  constructor(
    private route: ActivatedRoute,
    private noteApiService: NoteApiService,
    private formBuilder: FormBuilder) {
  }

  ngOnInit() {
    this.getNote();
  }

  getNote(): void {
    const id = +this.route.snapshot.paramMap.get('id');

    // Request the service for the record and create the form
    this.noteApiService.get(id).subscribe(value => this.createForm(value));
  }

  createForm(note: Note) {
    // use the FormBuilder to create a FormGroup
    this.editForm = this.formBuilder.group(note);

    // Watch for changes in the form
    this.editForm.valueChanges.subscribe(value => {
      console.log(value);
    })
  }
  
}
```

note.component.html
```html
<h2>Note details</h2>
<a routerLink="/notes">Back to the overview</a>

<form [formGroup]="editForm" *ngIf="editForm">
  <label>
    id:
    <input type="number" formControlName="id" readonly>
  </label>
  <label>
    Name:
    <input type="text" formControlName="name">
  </label>
</form>

```

Can you do this for groups? 

## 3.4 Create/Update a note
Great we can display data, but for displaying we need data. 
Creating and updating is almost the same so we will do that in the same way.

First we will extend our service with two methods, create and update.
Next we will create a new button on the list for a new note.
And finally we add a save button to save the data.

Let's create the create and update methods. Add these to your existing service.

note-api.service.ts
```typescript
 
  public create(note: Note): Observable<Note> {
    return this.httpClient.post<Note>(this.API_URL, note);
  }

  public update(note: Note): Observable<Note> {
    return this.httpClient.put<Note>(this.API_URL + '/' + note.id, note);
  }
```

Add a new button to the note list. 

note-list.component.html
```html
<h2>Notes overview</h2>
<a routerLink="/note/0">new note</a>
<ul>
  <li *ngFor="let note of notes" routerLink="/note/{{note.id}}">{{note.name}}</li>
</ul>
```

Add a save button to the note. Just add this line to the end off the file.

note.component.html
```html
<button (click)="save()">save</button>
```

Next we need to implement the save, but also need to check if the id=0 to detect a new note.

note.component.ts
```typescript
   ...
   
   getNote(): void {
       const id = +this.route.snapshot.paramMap.get('id');
       if (id === 0) {
         // Create new note
         const note = new Note();
         note.id = 0;
         note.name = '';
         this.createForm(note);
       } else {
         // Retrieve the data
         this.noteApiService.get(id).subscribe(value => this.createForm(value));
       }
   }
   
   ...
   
   save() {
       const note = <Note> this.editForm.getRawValue();
       if (note.id === 0) {
         this.noteApiService.create(note).subscribe(value => {
           console.log('Created new node', value);
         });
       } else {
         this.noteApiService.update(note).subscribe(value => {
           console.log('Updated existing node', value);
         });
       }
   }

```

Did you notice the console.log after a succesfull create or update?
We probably want to go back to the list instead.

Let's change that replace the console.log statements or put this line below the console.log

note.component.ts
```typescript
this.router.navigate(['/notes']);
```

As we did not use router before we also add it to the constructor

note.component.ts
```typescript

...

import {ActivatedRoute, Router} from "@angular/router";

...

constructor(
    private route: ActivatedRoute,
    private router: Router,
    private noteApiService: NoteApiService,
    private formBuilder: FormBuilder) {
  }

```

# 4 Running outside of docker-compose

Often we want to run angular or spring without docker-compose.
So we can test our worker faster without restarting the whole docker.

First we need to know we can manage containers individually.

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
Create an application-dev.properties in the same folder as application.properties with the following content:

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
In Angular 6 has something called environments and give us the same behaviour as profiles in Spring Boot.
Let's add an development environment configuration to the angular.json file. 

Angular 6 contains two environments by default:
* environment.ts
* environment.prod.ts

Environment.ts can be seen as the default development profile.
When deploying/building with prodcution profile it will overwrite environment with the environmet.prod.ts.

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

We now need to adjust our services (note.service.ts and group.service.ts).
So it will use the api_endpoint specified in the environment file.

In the note.service.ts file change the following line
```java
private API_URL = 'http://localhost:8090/notes';
```
into
```java
private API_URL = environment.api_endpoint + '/notes';
```

Do the same for the group.service.ts, but let it end on '/groups' instead of '/notes'.

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

`npm run start` will cal; start in the scripts node and run `ng serve`.


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

# 5 MapStruct

## What is MapStruct?
MapStruct is a code generator based on annotations and an interface.
Often there is a need to copy properties from one class to the other.
If there are many Data Transfer Objects (DTO) in your project this is an big helper. 
These interfaces are called mappers.

## Setup maven

Add an property to your pom.xml with the version of mapstruct. Just to be sure also check if you have java.version property.

```xml
<properties>
    ...
    <java.version>1.8</java.version>
    <org.mapstruct.version>1.2.0.Final</org.mapstruct.version>
    ...
</properties>
```

Next add the maven dependency for MapStruct.

```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-jdk8</artifactId>
    <version>${org.mapstruct.version}</version>
</dependency>
```

We also need to configure an annotationProcessor. This will make MapStruct run during compile time.
When you configure one annotationProcessor you also need to add lombok.
Root cause is that maven does not scan the classpath any more if paths are defined.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.5.1</version>
    <configuration>
        <source>${java.version}</source>
        <target>${java.version}</target>
        <annotationProcessors>
            <annotationProcessor>lombok.launch.AnnotationProcessorHider$AnnotationProcessor</annotationProcessor>
            <annotationProcessor>org.mapstruct.ap.MappingProcessor</annotationProcessor>
        </annotationProcessors>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.18.0</version>
            </path>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>${org.mapstruct.version}</version>
            </path>
        </annotationProcessorPaths>
        <compilerArgs>
            <arg>-Amapstruct.defaultComponentModel=spring</arg>
        </compilerArgs>
    </configuration>
</plugin>
```

The compiler argument is needed for MapStruct it will add @Component to the generated classes so we can AutoWire it.

## Creating a NoteMapper

Before creating the NoteMapper we first need an DTO which can be filled. This is our current Note Entity.

Note.java

```java
package com.example.nextnote.note;

import javax.persistence.*;

import com.example.nextnote.group.Group;

import lombok.Data;

@Data
@Entity
@Table(name = "note")
public class Note {
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "groupId")
	private Group group;

	private String name;
}
```

Let's create the NoteDto beside Note.java (in the same package).

NoteDto.java
```java
package com.example.nextnote.note;

import lombok.Data;

@Data
public class NoteDto {
	private Long id;
	private String name;
}
```

And now the mapper which will copy the properties from the entity to the dto and back.

NoteMapper.java
```java
package com.example.nextnote.note;

import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.MappingTarget;
import org.mapstruct.Mappings;

@Mapper
public interface NoteMapper {
	@Mappings({
			@Mapping(target = "id", ignore = true)
	})
	void toEntity(NoteDto noteDto, @MappingTarget Note note);

	NoteDto toDto(Note note);

}
```

That's it. This will create an mapper class which copies all the properties and will ignore the id when it goes back to an entity
You can even view the generated source in the target/generated-sources folder.

NoteMapperImpl.java
```java
package com.example.nextnote.note;

import javax.annotation.Generated;
import org.springframework.stereotype.Component;

@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2018-09-06T11:51:35+0200",
    comments = "version: 1.2.0.Final, compiler: javac, environment: Java 1.8.0_181 (Oracle Corporation)"
)
@Component
public class NoteMapperImpl implements NoteMapper {

    @Override
    public void toEntity(NoteDto noteDto, Note note) {
        if ( noteDto == null ) {
            return;
        }

        note.setName( noteDto.getName() );
    }

    @Override
    public NoteDto toDto(Note note) {
        if ( note == null ) {
            return null;
        }

        NoteDto noteDto = new NoteDto();

        noteDto.setId( note.getId() );
        noteDto.setName( note.getName() );

        return noteDto;
    }
}
```

As you can see above the class is annotated with @Component. This allows us to use @AutoWired in spring.

Example: 

```java
// The preferred way is by constructor, but this is easier for the example
@AutoWired
private NoteMapper noteMapper;

public NoteDto doSomething(Note note) {
	NoteDto noteDto = this.noteMapper.toDto(note);
	// More code
	return noteDto;
}

```