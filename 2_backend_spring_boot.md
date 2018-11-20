# 2 Backend boot
So now we have everything setup we can start working on the backend. The backend will be created with spring boot.
Spring boot comes with a lot of helper libraries to make our live easier.
But before we can do that we first need to setup the database.

We will do that and more in the application.properties 

## 2.1 Application properties
application.properties is located in the resource folder of the project (backend/src/main/resources/application.properties) and probably is empty right now.
In Spring Boot it is possible to have a properties file or an yaml file. We will be using the properties.

Also it is possible to have different properties for different environments. We will be using the default one.

We need to make Spring Boot aware of our datasource, so let's add these lines:

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
Next you can run the same command as before from within the `docker` folder:

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

The important part of this error is `please add migrations or check your Flyway configuration` let's do that.

By default flyway checks `resources/db/migration`. 
You already have resources that is where your `application.properties` is.
Create a folder ```db``` in the resources folder. Create a `migration` folder in the just created db folder.

Now we need to add an sql file in the `migration` folder.
Create a file called `V00001__initial.sql` in the migration folder.
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

Create a file called `V00002__group.sql` in the migration folder.

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

Now we have a backend running and a database we can continue with the entities.

## 2.3 Entities and lombok

Entities are the mapping between your database tables and the java code.
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

> Watch out: @Data generates also toString which creates a string off all properties, this often is to much.
 
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

Create a new file called `NoteRepository.java` in the `note` folder.

```java
package com.example.nextnote.note;

import org.springframework.data.jpa.repository.JpaRepository;

public interface NoteRepository extends JpaRepository<Note, Long> {
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
So go ahead and make the GroupRepository interface in the `group` folder. 

## 2.5 REST security
At this stage we are not going to implement a fully featured security model.
But as we added the security starter from Spring we need to allow everything for easy testing.

Create a folder `config` and in that folder a file called `WebSecurityConfig.java` with the following contents:

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

Because we are going to use REST services we will encounter CORS exceptions.
To prevent this we need to prepare Spring Boot for it.

So we add another config class called `WebConfig.java` with the following contents:

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

## 2.6 REST controllers
Now we can display our data, we should make it available to the frontend. This will be done by restful service. 
In Spring these are controllers of data. And for rest we will use the annotation @RestController for it. 

Let's create `NoteController.java` in the folder `note` and add the following contents:

```java
package com.example.nextnote.note;

import java.util.List;
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
	public List<Note> all() {
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
> But again, for simplicity reasons I left it out

Now go ahead and do the same for groups.

After this you can build and restart the docker containers.

> use `./build.sh && docker-compose up` in the docker folder to start it 

Now you can test your restful services with postman, soapui or any other tool.
A quick test could be to run it in your browser with the following url:
> http://localhost:8090/notes

The database is still empty so it would return an empty array.

Time to continue with the angular part.